# build stage
FROM node:22-alpine AS build

WORKDIR /app

COPY package.json pnpm-lock.yaml ./

# Install pnpm
# Pinned to a major on purpose. pnpm@latest resolves to 10+, which promotes
# ERR_PNPM_IGNORED_BUILDS from a warning to a hard install failure, and this
# project does not declare `onlyBuiltDependencies` in package.json. Pinning also
# keeps the image build reproducible.
RUN npm install -g pnpm@9

RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm run generate

# production stage
FROM nginx:stable-alpine AS production
COPY --from=build /app/.output/public /usr/share/nginx/html

# Copy the nginx configuration file
COPY ./nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
