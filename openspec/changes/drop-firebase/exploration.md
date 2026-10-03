# Exploration: drop-firebase

> Phase: `sdd-explore` | Change: `drop-firebase` | Date: 2026-10-02
> Read-only investigation. No application, config, env, or CI file was modified.

## TL;DR — Three findings that reframe the change

1. **The system is genuinely MULTI-TENANT, and its identity is already decoupled from Firebase.**
   `entities.UserID` is `type UserID string` (`api/pkg/entities/user.go:10`) and is the primary key of
   `entities.User` (`user.go:75`). It is *not* a hardcoded `"user"`. Every `UserID` field carries
   `example:"WB7DRDWrJZRGbYrv2CKGkqbzvqdC"` — a 28-char Firebase UID. Removing Firebase Auth is
   therefore **not** a re-keying migration across 14 tables. It is a swap of the **proof mechanism**
   (who can mint that string), not of the identity data.
2. **A Firebase-free auth path already exists and is already the primary one for machines.**
   `APIKeyAuth` (`api/pkg/middlewares/api_key_auth_middleware.go:14`) resolves an `AuthContext`
   purely from Postgres via `gormUserRepository.LoadAuthContext`
   (`api/pkg/repositories/gorm_user_repository.go:157-189`). `PhoneAPIKeyAuth`
   (`phone_api_key_auth_middleware.go:14`) and `BearerAPIKeyAuth`
   (`bearer_api_key_auth_middleware.go:14`) do the same. Android, the event queue, and all
   third-party API clients already authenticate this way. Only the web dashboard depends on a
   Firebase-issued bearer token.
3. **The Firebase-free push path already ships in production code.**
   `entities.Phone.NotificationTransport()` (`api/pkg/entities/phone.go:78-105`) inspects
   `FcmToken`: a URL-shaped value selects `NotificationTransportHTTP`, served by
   `HTTPNotificationSender` (`api/pkg/services/http_notification_sender.go:32`). That sender needs
   no Firebase at runtime — it only reuses `messaging.Message` as a JSON envelope shape.

The correct framing is: **"remove the Firebase SDK and the Firebase Hosting deploy target"**,
not **"migrate the datastore"** (Postgres is already primary) and not **"rebuild the identity model"**
(the identity model is fine).

---

## Current State

### Where Firebase actually lives (verified, not assumed)

`api/` — **10 Go files import `firebase.google.com/*`**, nothing else:

| File | Lines | Firebase API used |
|---|---|---|
| `api/pkg/di/container.go` | 58-59, 429-438, 479-487, 546-566, 586-590 | `firebase.App`, `auth.Client`, `messaging` |
| `api/pkg/middlewares/bearer_auth_middleware.go` | 8, 34 | `auth.Client.VerifyIDToken` |
| `api/pkg/services/user_service.go` | 10, 501-511 | `auth.Client.DeleteUser` |
| `api/pkg/services/marketting_service.go` | 10, 22, 70, 93 | `auth.Client.GetUser`, `auth.UserRecord` |
| `api/pkg/services/fcm_client.go` | 6, 15, 29 | `messaging.Message` (type leak) |
| `api/pkg/services/emulator_fcm_client.go` | 11, 54 | `messaging.Message` (type leak) |
| `api/pkg/services/http_notification_sender.go` | 14, 67-130 | `messaging.Message` (type leak only) |
| `api/pkg/services/phone_notification_service.go` | 10, 95, 148, 194 | `messaging.Message` construction |
| `api/pkg/services/phone_notification_service_test.go` | 10 | `messaging.Message` |
| `api/pkg/services/http_notification_sender_test.go` | 15 | `messaging.Message` |

`firebase.google.com/go v3.13.0+incompatible` is a **direct** require (`api/go.mod:9`).
`cloud.google.com/go/firestore v1.24.0` at `api/go.mod:84` is an **unused indirect** dep — confirmed zero
imports across the repo. It is dead weight, not a Firestore migration risk.

`web/` — Firebase used for **Auth only**. No `firebase/messaging` import anywhere; `getMessaging` is never
called. `nuxt.config.ts:89-90` transpiles only `firebase/app` and `firebase/auth`.
Five files import `firebase/auth`:
- `web/app/plugins/firebase.client.ts` (47 lines) — app init + `onAuthStateChanged` bridge
- `web/app/stores/auth.ts:2` — `import type { User as FirebaseUser }`
- `web/app/components/FirebaseAuth.vue` (533 lines) — Google/GitHub popup OAuth, email+password,
  password reset, profile update, ~15 Firebase error codes mapped at lines 227-272
- `web/app/layouts/default.vue:5,50-51` — `getAuth()`, `auth.currentUser?.getIdToken()`
- `web/app/components/MessageThreadHeader.vue:3,71-72` — `getAuth()`, `signOut()`

`android/` — FCM only, never Firebase Auth:
- `android/app/build.gradle.kts:59,61,62` — `firebase-bom:34.16.0`, `firebase-analytics`, `firebase-messaging`
- `android/app/build.gradle.kts:3` — `id("com.google.gms.google-services")`
- `android/app/google-services.json` (47 lines)
- `android/app/src/main/AndroidManifest.xml:70-76` — `MyFirebaseMessagingService` + `MESSAGING_EVENT` filter
- `android/app/src/main/java/com/httpsms/FirebaseMessagingService.kt` (451 lines) —
  `class MyFirebaseMessagingService : FirebaseMessagingService()`; `onMessageReceived` reads
  `KEY_HEARTBEAT_ID` / `KEY_MESSAGE_ID`; `onNewToken` POSTs to `/v1/phones/fcm-token`

`tests/` — **already Firebase-free**. `tests/docker-compose.yml:115` feeds
`FIREBASE_CREDENTIALS` from a shell var, and `tests/generate-firebase-credentials.sh` mints a **fake**
service account (throwaway RSA key, `token_uri: http://wiremock:8080/token`) purely so the SDK can sign a
JWT nobody validates. Push goes through `FCM_ENDPOINT=http://wiremock:8080` (`tests/.env.test:10`) →
`EmulatorFCMClient` → `tests/wiremock/mappings/fcm-send.json`.

`.github/workflows/web.yml:85-92` — **a fourth Firebase surface not in the original inventory**:
`FirebaseExtended/action-hosting-deploy` publishes `./web` to Firebase Hosting `channelId: live`
alongside a parallel Cloudflare Pages deploy (lines 77-83).

### The single hard boot-time seam

`api/pkg/di/container.go:209`

```go
app.Use(middlewares.BearerAuth(container.Logger(), container.Tracer(), container.FirebaseAuthClient()))
```

This is unconditional, global, and evaluated inside `App()`. `FirebaseAuthClient()`
(`container.go:479-487`) → `FirebaseApp()` (`container.go:429-438`) →
`firebase.NewApp(ctx, nil, option.WithCredentialsJSON(container.FirebaseCredentials()))`.
`FirebaseCredentials()` (`container.go:586-590`) is just `[]byte(os.Getenv("FIREBASE_CREDENTIALS"))`.

**Consequence: `FIREBASE_CREDENTIALS` must contain valid service-account JSON for the API to boot at
all** — even in emulator mode where `FCMClient()` returns early at `container.go:551-558` and never
touches Firebase, and even though the E2E suite supplies only a self-signed fake. `api/.env.docker:23`
and `api/.env.production:13` both ship it empty, which is why the local stack cannot start unconfigured.
This is the one-line seam that makes the whole removal tractable.

### `FIREBASE_CREDENTIALS` is doing three unrelated GCP jobs

| Consumer | Line | Actually needs |
|---|---|---|
| `FirebaseApp()` → auth + messaging | `container.go:433` | Firebase |
| `CloudTasksClient()` | `container.go:493` | Google Cloud Tasks (GCP, not Firebase) |
| GCS `storage.NewClient` | `container.go:1676` | Google Cloud Storage (GCP, not Firebase) |

So "drop Firebase" cannot delete the env var — it must be **renamed** (e.g. `GOOGLE_SERVICE_ACCOUNT_JSON`)
and Firebase removed from its consumers. `GCS_BUCKET_NAME` is empty in `tests/.env.test`, so the GCS
branch is already inert there.

---

## Affected Areas

### `api/` — identity
- `api/pkg/middlewares/bearer_auth_middleware.go:42-45` — builds `entities.AuthContext` from
  `token.Claims["email"].(string)` and `token.Claims["user_id"].(string)`. **Unchecked type assertions**:
  a token missing either claim panics the request goroutine instead of returning 401. Pre-existing bug
  that any replacement should fix.
- `api/pkg/middlewares/api_key_auth_middleware.go:14-38` — the Firebase-free sibling. Note line 24
  rejects any key with a `pk_` prefix, delegating phone-scoped keys to `PhoneAPIKeyAuth`.
- `api/pkg/middlewares/bearer_api_key_auth_middleware.go:22` — treats `Authorization: Bearer <api-key>`
  as an API key; strips the scheme and hits Postgres. **Already tolerates a non-Firebase bearer token.**
- `api/pkg/middlewares/authenticated_middlesare.go:21-35` — downstream gate; reads only
  `c.Locals(ContextKeyAuthUserID)`, provider-agnostic.
- `api/pkg/repositories/gorm_user_repository.go:157-189` (`LoadAuthContext`) and `:209+` (`LoadOrStore`)
  — the whole identity store. `LoadOrStore` creates a `User` keyed by `authUser.ID` with a generated
  `uk_`-prefixed API key (`gorm_user_repository.go:209-232`). Postgres-only, Firebase-free.
- `api/pkg/services/user_service.go:501-511` — `DeleteAuthUser`, the **only** Firebase *write* from the
  API. Sole caller: `api/pkg/listeners/user_listener.go:164` on `user.account-deleted`.
- `api/pkg/services/marketting_service.go:70` — `authClient.GetUser(userID)` to read `Email`/`DisplayName`
  for Plunk. `DisplayName` is **not stored in Postgres** (`entities.User` has `Email`, no name column —
  `user.go:73-91`). Losing Firebase Auth means losing `DisplayName` unless a column is added.

### `api/` — push
- `api/pkg/services/fcm_client.go:11-16` — `FCMClient.Send(ctx, message *messaging.Message, phoneID uuid.UUID)`.
  **The interface itself leaks the Firebase SDK type.** No provider swap is possible without changing
  this signature.
- `api/pkg/services/phone_notification_service.go:95-100, 148-156, 194, 217` — constructs `messaging.Message`
  and assigns `message.Token = strings.TrimSpace(*phone.FcmToken)`.
- `api/pkg/entities/phone.go:16` — `FcmToken *string` is a **polymorphic field**: Firebase registration
  token *or* HTTPS adapter URL. Load-bearing for both transports. Renaming it is a breaking API change
  (`web/shared/types/api.ts:250`, `:539-553`; `android/.../HttpSmsApiService.kt:263` posts `fcm_token`).
- `api/pkg/di/container.go:546-584` — `FCMClient()` factory + `PhoneNotificationClients()` map keyed by
  `entities.NotificationTransport`.

### `web/`
- `web/app/stores/auth.ts:36-47` — `setAuthHeader(await firebaseUser.getIdToken())`. **Note:** `loadUser()`
  (`:50-53`) never calls `setApiKey`; only `updateUser()` (`:72`) and `rotateApiKey()` (`:91`) do. So on a
  fresh page load the dashboard holds **only** the Firebase bearer token until the user saves settings.
  The web is genuinely Firebase-dependent today.
- `web/app/composables/useApi.ts:26-31` — sends both `Authorization: Bearer` and `x-api-key` when set.
- `web/nuxt.config.ts:132-138` — 7 `firebase*` runtime keys; `firebaseMessagingSenderId`,
  `firebaseStorageBucket`, `firebaseMeasurementId` are **dead** (no messaging/analytics/storage in web).
- `web/.env.docker`, `web/.env.production` — 7 `FIREBASE_*` keys each.
- `web/firebase.json` (12), `web/.firebaserc` (5), `web/package.json:32` (`firebase: ^12.13.0`).

### `android/`
- `android/.../ui/login/LoginViewModel.kt:125-130` — `login()` **refuses to proceed** when
  `Settings.getFcmToken(context) == null`, firing `onFcmTokenMissing`.
- `android/.../LoginActivity.kt:52` — login button is additionally gated on
  `isGooglePlayServicesAvailable()` (`:148-150`). Both gates must be removed for an FCM-free Android client.
- `android/.../MainActivity.kt:170-201` — re-uploads the cached FCM token every 24h; bails at `:170`
  if the token is null.
- `android/.../FirebaseMessagingService.kt:104-120` — `onNewToken` → `Settings.setFcmTokenAsync` +
  `HttpSmsApiService.updateFcmToken` for SIM1/SIM2.

### `tests/` and CI
- `tests/docker-compose.yml:115`, `tests/.env.test:10`, `tests/generate-firebase-credentials.sh` (31),
  `tests/wiremock/mappings/fcm-send.json`, `tests/helpers_test.go:520,638-650` (`findFCMRequests`,
  `waitForFCMPush`).
- `.github/workflows/api.yml:26-29` — the only reason the fake credential exists. **Never runs
  `go test ./...`** on `api/` (208 test funcs, 0 executed in CI); it runs only
  `go test -tags integration ./pkg/handlers` (line 82-84) plus the separate `tests/` module (line 87-88).
- `.github/workflows/web.yml:85-92` — Firebase Hosting deploy.

---

## Answers to the Six Questions

### Q1 — Is the system single-tenant or multi-tenant?

**Multi-tenant, with identity already independent of Firebase.**

Evidence:
- `api/pkg/entities/user.go:10` — `type UserID string`. A bare string alias, no Firebase type.
- `api/pkg/entities/user.go:75` — `ID UserID \`gorm:"primaryKey;type:string;"\``, `example:"WB7DRDWrJZRGbYrv2CKGkqbzvqdC"`.
  28-char Firebase UID format, and the same `example` tag is repeated on all 14 entity `UserID` fields
  (`message.go:90`, `contact.go:69`, `phone.go:15`, `webhook.go:13`, `heartbeat.go:15`, `discord.go:12`,
  `message_thread.go:16`, `message_send_schedule.go:22`, `phone_api_key.go:17`, `phone_notification.go:25`,
  `billing_usage.go:12`, `heartbeat_monitor.go:13`, `integration_3cx.go:12`).
- `tests/seed.sql:5-38` — three distinct users (`test-user-id`, `rotate-test-user-id`, `system-user-id`),
  proving multi-tenancy end to end.
- 747 non-test `UserID` references across 150 Go files — pervasive, but all *reads/writes of a string*,
  never of a Firebase type.

Correction to a prior assumption: there is **no hardcoded `"user"`** in the codebase. The nearest thing is
the synthetic service identity `EVENTS_QUEUE_USER_ID=system-user-id` (`api/.env.docker:19`,
consumed at `container.go:508`), which is a real seeded user row, not a global tenant.

**Implication:** the ID→row mapping survives verbatim. `LoadOrStore`
(`gorm_user_repository.go:209-232`) will find the existing `User` by the unchanged `authUser.ID`.
**There is no data migration.** The only thing that changes is *who can present a bearer token that
resolves to that string*.

### Q2 — Auth replacement options

#### (a) Keep Firebase Auth as a pure JWT verifier; drop all Firebase client SDK / messaging

- **api/**: Replace `auth.Client.VerifyIDToken` (`bearer_auth_middleware.go:34`) with offline verification —
  fetch JWKS from `https://www.googleapis.com/robot/v1/metadata/x509/securetoken@system.gserviceaccount.com`,
  verify RS256 with `github.com/golang-jwt/jwt/v5` (already a **direct** dep, `api/go.mod:29`), assert
  `aud == FIREBASE_PROJECT_ID`, `iss == https://securetoken.google.com/<project>`, `exp`, and read
  `sub` (not the legacy `user_id` claim) plus `email`. Drops `firebase.google.com/go` from `api/` entirely.
  `DeleteAuthUser` (`user_service.go:501-511`) is deleted; the `user_listener.go:164` call site too.
  `MarketingService.CreateContact` (`marketting_service.go:70`) needs `entities.User.Email` from Postgres
  instead of `authClient.GetUser`; `DisplayName` is **unavailable** — accept losing `firstName`/`lastName`
  in Plunk events, or add a `display_name` column.
- **web/**: **Unchanged.** Still needs the Firebase Web SDK to sign in. This option only removes *messaging*,
  not *auth*, from web.
- **android/**: **Unchanged** — never used Firebase Auth.
- **tests/**: `generate-firebase-credentials.sh` can be deleted once `CloudTasksClient` and GCS are
  repointed; `api.yml:26-29` goes away.
- **Effort**: Medium, ~350-450 changed lines total, 2-3 PRs.
- **Risk**: Low. It also lets you fix the unchecked type assertions.
- **Existing users**: **No password reset.** Nothing about credentials changes.

#### (b) Self-hosted sessions/JWT issued by the Go API, backed by Postgres

- **api/**: New code — session/refresh-token entity + `AutoMigrate` line in `container.go:286-422`,
  repository interface + `gorm*` implementation, session service, `/v1/auth/*` handlers, and a
  `TokenVerifier` interface that `BearerAuth` (`bearer_auth_middleware.go:16`) depends on.
- **web/**: `FirebaseAuth.vue` (533 lines) is fully rewritten: Google/GitHub OAuth must be reimplemented
  against their REST endpoints, or delegated to an IdP (see (c)). The ~15 Firebase error-code mappings
  (`:227-272`) all have to be re-derived. `firebase.client.ts` (47), `auth.ts` (120), `default.vue:50-51`,
  `MessageThreadHeader.vue:71-72` all change.
- **android/**: Unaffected.
- **tests/**: E2E needs a session-creation step in `helpers_test.go`.
- **Effort**: High, ~1400-1800 changed lines, 5-7 PRs.
- **Risk**: Medium-High. Reimplementing Google/GitHub OAuth + refresh + revocation by hand is where real
  projects get security bugs. Passwords must be hashed locally (bcrypt/argon2) — a new attack surface.
- **Existing users**: **Password reset required** for email/password users. Firebase's bcrypt hashes are
  not exportable through the Admin SDK. Google/GitHub OAuth users could be migrated silently only via a
  custom claim on the existing Firebase users and a one-time exchange — otherwise they re-auth.

#### (c) Third-party IdP (Auth0 / Keycloak / Supabase Auth / Clerk)

- **api/**: identical work to (b) — same `TokenVerifier` seam, but JWKS issuer URL and claim mapping
  (`sub` → `UserID`, `email` → `Email`) instead of local sessions. **Supabase Auth is the cheapest fit**:
  it issues HS256 JWTs signed with a shared secret the Go API verifies directly, matching the
  self-hosted JWT code the notification sender already uses
  (`http_notification_sender.go:116-123`), and it can import Firebase users only via manual re-registration.
  Keycloak is the best self-hosted option but heaviest to operate. Clerk/Auth0 are SaaS lock-in the project
  is trying to leave.
- **web/**: same rewrite as (b); provider SDK is drop-in for the login component, so `FirebaseAuth.vue`
  shrinks rather than being hand-rewritten.
- **android/**: Unaffected.
- **Effort**: Medium-High, ~1100-1500 lines.
- **Risk**: Medium. Vendor dependency, but the token *verification* logic is small.
- **Existing users**: **Password reset required** unless the IdP import path is used.

**Recommendation: (a) now, (b)/(c) later as a separate change.**
Option (a) is the only one with a clean user story, and it is the only one that removes the
`firebase.google.com/go` dependency. Splitting "remove the SDK" from "replace the IdP" keeps every slice
reversible. Do **not** attempt (b) and (c) inside `drop-firebase`.

### Q3 — FCM replacement

#### Web push — **NON-ISSUE. There is none.**
Verified: no `firebase/messaging` import in `web/`, no `getMessaging()` call, no service worker for push.
`nuxt.config.ts:132-138` declares `firebaseMessagingSenderId` but nothing consumes it. VAPID/Web Push is
**not** a migration target — it is a net-new feature. Skip it.

#### Android push — the honest verdict

**(i) Drop the Firebase SDK, keep the FCM wire protocol.** ✅ Feasible, recommended.
`FCM HTTP v1` is `POST https://fcm.googleapis.com/v1/projects/{project}/messages:send` with
`Authorization: Bearer <OAuth2 access token scoped to firebase.messaging>` and a
`{"message": {"token": ..., "data": ..., "android": {...}}}` body. The repo **already builds that exact
envelope** in `emulator_fcm_client.go:34-46` and `http_notification_sender.go:126-130`, and
`golang.org/x/oauth2 v0.36.0` + `google.golang.org/api v0.295.0` are already in the module graph
(`api/go.mod:202,60`) — promoting them to direct requires no new third-party surface. Writing a
`FCMHTTPv1Client` that satisfies the existing `FCMClient` interface is ~120 lines.

**But this does not remove Firebase.** It removes the *SDK* while keeping Google as the transport. It is
honest only if the goal is "drop the dependency", not "drop Google".

**(ii) True Android push without FCM — A TRAP. Do not attempt it in `drop-firebase`.**

- **UnifiedPush** is not a server protocol. It is a *client-side bridge* that lets an app consume
  another installed app's push channel (e.g. a user's Electric Eel / Tusky / FluffyChat relay). It
  **requires a third-party app to be installed on the device**. There is no self-hosted UnifiedPush
  server that reliably delivers to arbitrary phones. Making it the sole wake-up path means asking every
  user to install an additional app — unacceptable for an SMS gateway app where a missed push = a
  missed SMS.
- **Self-hosted MQTT** is not an Android *push* channel. A phone in Doze with no MQTT connection receives
  nothing until it wakes — and this app's entire purpose is waking a **sleeping** phone to send an SMS.
  It needs a connection that survives Doze without the app running, which on stock Android means FCM.
- **Persistent foreground service / `WorkManager` polling** trades SMS latency (seconds → minutes) and
  battery for reliability. `StickyNotificationService` (`android/app/src/main/AndroidManifest.xml:60-64`)
  already runs with `foregroundServiceType="remoteMessaging"`; leaning on it further risks Android
  background-execution limits and Play Store policy review.
- **The heartbeat mechanism makes this fatal.** `phone_notification_service.go:82-119`
  (`SendHeartbeatFCM`) exists specifically to *wake a phone that has not checked in*. Replacing the wake-up
  channel with something that requires the phone to be awake removes the feature's reason to exist.
- **The `fcm_token` field is doubly load-bearing.** `phone.go:16` stores either an FCM registration token
  or an HTTPS adapter URL, discriminated by `NotificationTransport()` (`phone.go:78-105`). Renaming or
  re-purposing it is a breaking change rippling into `web/shared/types/api.ts:250,539-553`,
  `android/.../HttpSmsApiService.kt:263`, `api/pkg/requests/phone_fcm_token_request.go:14-19`, and
  `tests/helpers_test.go:124,201`. **Leave it named `fcm_token` and leave its semantics alone.**

**Verdict: Android push without FCM is a product-level decision (degrade to polling, or require a
relay app), not an engineering one. It is out of scope for `drop-firebase`.**

#### Also in scope for "remove Firebase": Firebase Hosting
`.github/workflows/web.yml:85-92` deploys to Firebase Hosting. A parallel Cloudflare Pages deploy already
runs at `:77-83`. Since `openspec/config.yaml:15` records Cloudflare as out of scope, removing the
Firebase Hosting step is a clean, self-contained deletion — but flag it to the user, because
`httpsms.com` may currently resolve there and `FIREBASE_AUTH_DOMAIN=httpsms.com` in
`web/.env.production` hints at that coupling.

---

## Approaches — Slicing Order

Chosen so the system stays runnable at every step and each slice fits the 400-line review budget.

| # | Slice | Files | ~Lines | PRs |
|---|---|---|---|---|
| **0** | **CI test gate (spike, unsized)** | `.github/workflows/api.yml`, triage | ? | ? |
| **1** | Rename `FIREBASE_CREDENTIALS` → `GOOGLE_SERVICE_ACCOUNT_JSON` | `container.go:586-590,493,1676`; `tests/docker-compose.yml:115`; `tests/.env.test`; `api.yml:26-29` | ~40 | 1 |
| **2** | Kill dead Firebase config in `web/` | `nuxt.config.ts:132-138`; `.env.docker`; `.env.production` | ~35 | 1 |
| **3a** | Introduce `entities.NotificationMessage`; migrate `fcm_client.go` + `emulator_fcm_client.go` | new entity; `fcm_client.go:11-16`; `emulator_fcm_client.go:34-65`; tests | ~190 | 1 |
| **3b** | Migrate `http_notification_sender.go` | `:67-130`; its test (466) | ~150 | 1 |
| **3c** | Migrate `phone_notification_service.go` | `:95-100,148-156,194,217`; its test (329) | ~130 | 1 |
| **4** | Add `FCMHTTPV1Client` (direct protocol, no SDK) | new ~120 + test ~120; `container.go:546-566` | ~260 | 1 |
| **5a** | `TokenVerifier` interface behind `BearerAuth` | `bearer_auth_middleware.go:16-49`; new iface + test; `container.go:209,479-487` | ~230 | 1 |
| **5b** | Offline Google-signed JWT verifier | new ~110 + test ~180 | ~290 | 1 |
| **6** | Delete `FirebaseFCMClient`, `DeleteAuthUser`, `MarketingService.authClient` | `fcm_client.go`; `user_service.go:501-511`; `user_listener.go:164`; `marketting_service.go:66-110`; `marketing_listener.go:48`; `container.go:1108-1117,1120-1134`; `go.mod:9` | ~220 | 1 |
| **7** | Delete `generate-firebase-credentials.sh` + its CI step | script (31); `api.yml:26-29` | ~40 | 1 |
| **8** | Remove Firebase Hosting deploy | `web.yml:85-92`; delete `firebase.json`, `.firebaserc` | ~20 | 1 |

**Total ≈ 1600 changed lines across 12 PRs.** Every slice is independently revertible.

### Slice 0 must come first — and it is a genuine unknown

`.github/workflows/api.yml` has never run `cd api && go test ./...` (208 test funcs). Adding it may
fail immediately. **No Go toolchain exists on this host** (`which go` → not found), so I cannot determine
whether the suite is green. Slices 1-9 will be implemented under `strict_tdd: true`
(`openspec/config.yaml:17`) with `cd api && go test ./...` as the gate — which means the gate itself
must be proven first.

Do **not** start implementation before Slice 0 reports a green baseline. This is the one place where I am
reporting a risk rather than a fact.

---

## Interface Seam — Exact Locations

**Seam 1 — Global middleware chain (auth provider swap, zero call-site churn).**
`api/pkg/di/container.go:209-210`. Two `app.Use` calls, both terminating in
`c.Locals(ContextKeyAuthUserID, authUser)`. Both call `c.Next()` on failure (never reject), so they form a
fallback chain. Deleting line 209 removes Firebase verification for every route while leaving
`APIKeyAuth` (`:210`) and all 15 `Register*Routes` calls (`:1430-1810`) untouched.
Also unused today but available: `BearerAPIKeyMiddleware()` (`:216-220`), `PhoneAPIKeyMiddleware()`
(`:222-226`), `AuthenticatedMiddleware()` (`:228-232`) — the latter two are wired per-route at
`:1430,1436,1442,1747,1773,1792`.

**Seam 2 — `FCMClient` interface.** `api/pkg/services/fcm_client.go:11-16`. Interface, three
implementations (`FirebaseFCMClient:19`, `EmulatorFCMClient:emulator_fcm_client.go:18`,
`HTTPNotificationSender:http_notification_sender.go:32,39` asserted via `var _ FCMClient`). Registered as
a transport-keyed map by `container.PhoneNotificationClients()` (`container.go:576-584`), consumed only by
`phone_notification_service.go:209-218`. **The interface is Firebase-typed and must be re-cut before any
provider swap** (slices 3a-3c). This is the textbook interface-seam pattern and it is already correctly
structured apart from that one type leak.

**Seam 3 — `firebase.NewApp` factory.** `container.go:429-438`. Replace `FirebaseAuthClient()` (`:479-487`)
and the `Messaging()` branch in `FCMClient()` (`:561-565`) with independent providers. `container.go:550`
is already an if/else factory on `FCM_ENDPOINT`, so the pattern is established.

**Seam 4 — `AuthContext` as the sole carrier.** `api/pkg/entities/auth_context.go:6-11`
(`ID`, `Email`, `PhoneAPIKeyID`, `PhoneNumbers`). Written in exactly 4 places —
`bearer_auth_middleware.go:47`, `api_key_auth_middleware.go:35`, `bearer_api_key_auth_middleware.go:33`,
`phone_api_key_auth_middleware.go:33` — read in `authenticated_middlesare.go:26` and
`handlers/handler.go:124-125`, then passed to 13 `requests/*.To*Params` methods. **The identity contract
is 11 lines wide.** Any auth replacement only has to produce an `AuthContext`; nothing downstream changes.

**Seam 5 — `NotificationTransport` dispatch.** `api/pkg/entities/phone.go:78-105`, resolved at
`phone_notification_service.go:200-215` against `container.PhoneNotificationClients()`
(`container.go:576-584`). This is the switch that lets FCM be dropped without touching the HTTP adapter path.

**Non-seam (be honest about this):** `entities.UserID` at 747 references across 150 files. Do **not** try
to abstract it. It is already a plain string and needs no change.

---

## Reversibility Per Slice

| Slice | Rollback |
|---|---|
| 0 (CI gate) | Remove the added step. Zero runtime effect. |
| 1 (env rename) | Accept **both** names during a deprecation window (`GoogleCredentials()` reads `GOOGLE_SERVICE_ACCOUNT_JSON` then falls back to `FIREBASE_CREDENTIALS`). Rollback = revert the reader. Set both in prod before merging. |
| 2 (dead web config) | Fully inert — remove the keys, deploy. Git revert restores them. No behavior change at any point. |
| 3a-3c (type migration) | Per-slice git revert. **Risk: `messaging.Message` and `entities.NotificationMessage` must produce byte-identical JSON**, or `HTTPNotificationSender` breaks its adapter contract and the Android app silently stops receiving pushes. Mitigation: golden-file test on the marshalled envelope in 3b. |
| 4 (HTTP v1 client) | Add a config flag to select between the new client and the SDK client; keep both for one release. Rollback = flip the flag. |
| 5a-5b (verifier) | Register both middlewares: Firebase `VerifyIDToken` first (sets locals on success), local verifier second. Currently `bearer_auth_middleware.go:37` already does exactly this — `c.Next()` on invalid token. Rollback = reorder two lines. |
| 6 (delete SDK) | Hardest to reverse. `go.mod:9` re-add + re-adding three constructors. **Only merge 5b into `main` and observe in production for ≥1 release before 6.** |
| 7 (test credential script) | Git revert. `api.yml` regenerates it. Harmless to keep a copy. |
| 8 (Hosting) | Re-add the workflow step. Note Cloudflare Pages already deploys in parallel, so a rollback is "point DNS back at Firebase Hosting". |

**Data rollback:** there is none to speak of — no slice mutates data. `User.ID` values never change
(Q1). The only irreversible step is credential deprovisioning, which happens *after* slices 5b and 6 are
proven, and is **out of scope for `drop-firebase`**.

---

## Risks

1. **Slice 0 is unsized and may be red.** 208 test funcs have never run in CI; no Go toolchain here to
   check. Do not begin implementation until this is green.
2. **`messaging.Message` → `entities.NotificationMessage` is a wire-format change** for the HTTPS adapter
   transport. Field names and nesting must match exactly or third-party adapters and the Android app
   silently lose wake-ups. Golden tests mandatory.
3. **`DisplayName` has no Postgres home.** `marketing_service.go:93-109` splits it into `firstName`/
   `lastName` for Plunk. Removing `authClient.GetUser` loses that data unless a `display_name` column is
   added — a schema change, which also means a `AutoMigrate` entry.
4. **Android login is gated on Play Services + FCM token** (`LoginViewModel.kt:129`,
   `LoginActivity.kt:52`). Any Android-side change must remove both gates or users cannot log in.
5. **Firebase Hosting may be serving production traffic** alongside the Cloudflare deploy. Removing
   `.github/workflows/web.yml:85-92` without checking DNS is an outage. Verify first.
6. **`api/.env.docker:23` / `api/.env.production:13` ship `FIREBASE_CREDENTIALS` empty.** Local boots
   already depend on the operator supplying it. Slice 1 must not make the new name mandatory without a
   fallback.
7. **`web/app/stores/auth.ts:50-53` never calls `setApiKey` on `loadUser`.** Any plan that assumes the
   dashboard can fall back to `x-api-key` after dropping the bearer token will break the first request
   after a page load. Fix `loadUser()` or introduce a cookie session.
8. **Latent panic in the current code.** `bearer_auth_middleware.go:43-44` uses unchecked type
   assertions on claims. Any token lacking `email` or `user_id` panics the handler. Fix during 5b.

---

## Recommendation

**Scope `drop-firebase` as "remove the `firebase.google.com/go` SDK, the Firebase Web SDK, and the
Firebase Hosting deploy — nothing else."** Specifically:

- **Adopt option (a)** for auth: offline verification of Google-signed ID tokens. It removes the Go SDK,
  fixes the claim-assertion panic, requires no password reset, and leaves `web/` untouched until a
  separate IdP change.
- **Adopt FCM HTTP v1 written directly** for push. It removes the Admin SDK while keeping the transport,
  and reuses an envelope shape the codebase already emits twice. Say plainly that Google remains the
  push transport.
- **Do NOT touch** `fcm_token` field name/semantics, the `web/` login UI, Android login gating, or the
  data model. Each of those is a separate change with its own risk profile.
- **Do NOT attempt** Android push without FCM. UnifiedPush requires a third-party installed app;
  MQTT and foreground-service polling cannot wake a Dozing phone, which is the exact thing
  `SendHeartbeatFCM` exists to do. That is a product decision, not an engineering one.

### Ready for Proposal

**Yes**, with two conditions the orchestrator must put to the user first:

1. **Confirm the goal is "remove the SDK/dependency", not "remove Google".** Option (a) + FCM HTTP v1 keeps
   Google as the auth issuer and push transport. A genuine Google exit requires the IdP change (Q2 b/c)
   and the Android push decision (Q3 ii) — both out of scope here.
2. **Confirm Firebase Hosting is safe to remove** (DNS check), given `web.yml` runs it in parallel with
   Cloudflare Pages and `.env.production` sets `FIREBASE_AUTH_DOMAIN=httpsms.com`.

Slice 0 (prove `cd api && go test ./...` is green) must land before slice 1.