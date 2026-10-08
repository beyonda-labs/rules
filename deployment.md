# Deployment rules

How a product of Beyonda Labs is packaged and deployed: one Docker image per product, holding its service and its
front, that runs on any machine without being configured for it.

## Rules

- **One image per product**, built from the `Dockerfile` of its service repo: the service serves the build of the
  front and answers the API under `/api` (`FRONT_DIRECTORY` and `API_PATH`, through `staticDirectory` and `apiPath` of
  express-components), so one process, one port and one origin answer both. No proxy is needed and CORS stays off.
- **The front is a dependency of the service**: `pnpm build` writes `dist/<front>/package.json`, which makes that
  folder the package `@beyonda-labs/<front>` with the build under `browser/` and no dependency. Jenkins publishes it
  like the libraries, and the service pins it to an exact version, so an image always carries a known front.
- **The front holds no address**: it calls `/api` on the origin that served it. `environment.production.ts` replaces
  `environment.ts` through `fileReplacements`, and in development the Angular dev server passes `/api` to the service
  (`proxy.conf.json`, stripping the prefix), so the app makes the same calls in both.
- **The image carries its defaults and nothing of a machine**: the paths of the data, the backups, the logs and the
  secrets point under `/data`, a volume, and the process runs as `node`. A missing JWT secret is generated once and
  kept in `SECRETS_PATH`; without admin credentials the first start creates the superadmin and prints them to the
  standard output, never to the log files. Moving the image takes no certificate, path or configuration file.
- **The registry certificate is a build secret**: the Verdaccio CA reaches the build as `--mount=type=secret` (from
  `NPM_CA_FILE` or `NODE_EXTRA_CA_CERTS`), never a layer of the image. Installs use `--frozen-lockfile`; the
  production dependencies come from `--prod --ignore-scripts` followed by `pnpm rebuild` of the native modules, built
  inside the image for its system.
- **A `node_modules` is never copied between machines**: `better-sqlite3`, `bcrypt` and any native module are built
  for one system and one version of Node.
- **The compose lives in `docker/` of the service repo**: `compose.yaml` runs the image and is the only file a target
  needs, `compose.build.yaml` adds the build and its secret, and `.env.example` names what a machine may change
  (`PORT`, `VERSION`, the admin, `TRUST_PROXY`). The published port is the one [ports.md](ports.md) reserves.
- **An image moves as a file**: `docker save <image> -o <image>.tar` on the machine that built it, `docker load` and
  `docker compose up -d` next to `compose.yaml` on the target.
- **Behind a proxy, the service trusts it**: when Caddy terminates HTTPS in front of the image, `TRUST_PROXY` names
  it, so the logs and the rate limits see each client by its own address.

## Example

```dockerfile
# Dockerfile of the service repo
FROM node:22-bookworm-slim AS base
RUN corepack enable
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./

FROM base AS build
RUN --mount=type=secret,id=npm_ca NODE_EXTRA_CA_CERTS=/run/secrets/npm_ca pnpm install --frozen-lockfile
COPY tsconfig.json tsconfig.build.json ./
COPY src ./src
RUN pnpm run build

FROM base AS production-dependencies
RUN --mount=type=secret,id=npm_ca NODE_EXTRA_CA_CERTS=/run/secrets/npm_ca \
    pnpm install --prod --frozen-lockfile --ignore-scripts && pnpm rebuild better-sqlite3 bcrypt

FROM node:22-bookworm-slim
ENV API_PATH=/api \
    FRONT_DIRECTORY=/app/node_modules/@beyonda-labs/document-builder-front/browser \
    DATABASE_PATH=/data/document-builder.db \
    SECRETS_PATH=/data/secrets.json
WORKDIR /app
COPY --from=production-dependencies /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
RUN mkdir /data && chown node:node /data
USER node
VOLUME /data
CMD ["node", "dist/index.js"]
```

```yaml
# docker/compose.yaml
name: document-builder
services:
  document-builder:
    image: document-builder:${VERSION:-latest}
    restart: unless-stopped
    ports:
      - "${PORT:-8084}:3100"
    volumes:
      - data:/data
volumes:
  data:
```
