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
- **The front builds for a Content Security Policy**: the service sends the front policy of express-components as a
  header, so nothing in `index.html` may be inline. The production build sets `inlineCritical: false`, whose stylesheet
  loader is an inline `onload` the policy refuses; `bey-config check` warns when it is missing and stops a release. A
  front that needs another source widens `frontContentSecurityPolicy` in `buildAppConfig`, and never with
  `'unsafe-inline'` in `script-src`.
- **The image carries its defaults and nothing of a machine**: the paths of the data, the backups, the logs and the
  secrets point under `/data`, a volume, and the process runs as `node`. A missing JWT secret is generated once and
  kept in `SECRETS_PATH`; without admin credentials the first start creates the superadmin and prints them to the
  standard output, never to the log files. Moving the image takes no certificate, path or configuration file.
- **The image answers for its health**: a `HEALTHCHECK` calls `/ready` with the `fetch` of Node, as the slim image
  has no curl, and the compose sets a `stop_grace_period` above the 10 seconds `beyStartServer` gives the requests in
  flight, so `docker stop` lets the process close its database before it is killed.
- **The registry certificate is a build secret**: the Verdaccio CA reaches the build as `--mount=type=secret` (from
  `NPM_CA_FILE` or `NODE_EXTRA_CA_CERTS`), never a layer of the image. Installs use `--frozen-lockfile`; the
  production dependencies come from `--prod --ignore-scripts` followed by `pnpm rebuild` of the native modules, built
  inside the image for its system.
- **A `node_modules` is never copied between machines**: `better-sqlite3`, `bcrypt` and any native module are built
  for one system and one version of Node.
- **The compose lives in `docker/` of the service repo**: `compose.yaml` runs the image and is the only file a target
  needs, `compose.build.yaml` adds the build and its secret, and `.env.example` names what a machine may change
  (`PORT`, `VERSION`, the admin, `TRUST_PROXY`, `FRONT_URL` and the mail account). The published port is the one
  [ports.md](ports.md) reserves.
- **An image moves as a file**: `docker save <image> -o <image>.tar` on the machine that built it, `docker load` and
  `docker compose up -d` next to `compose.yaml` on the target.
- **Behind a proxy, the service trusts it**: when Caddy terminates HTTPS in front of the image, `TRUST_PROXY` names
  it, so the logs and the rate limits see each client by its own address, and the refresh cookie goes out `Secure`,
  since express-components sets it only on a request it knows came over HTTPS. HSTS is the proxy's: Caddy adds
  `Strict-Transport-Security`, which the service never sends.
- **The mail account belongs to the machine, never to the image**: the image keeps the log transport of
  express-components as its default, so it starts anywhere. A deployment sets `MAIL_TRANSPORT=smtp`, `SMTP_USER`,
  `SMTP_PASSWORD` (an app password, never the account's own) and `MAIL_FROM` in the `.env` next to `compose.yaml`,
  and `FRONT_URL` to the public address behind the proxy, since the image cannot know the address the mailed links
  must open. The log transport writes every link, token included, to the log, so no deployment keeps it.

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
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD ["node", "-e", "fetch(`http://127.0.0.1:${process.env.PORT}/ready`).then(r => process.exit(r.ok ? 0 : 1), () => process.exit(1))"]
CMD ["node", "dist/index.js"]
```

```yaml
# docker/compose.yaml
name: document-builder
services:
  document-builder:
    image: document-builder:${VERSION:-latest}
    restart: unless-stopped
    stop_grace_period: 15s
    ports:
      - "${PORT:-8084}:3100"
    volumes:
      - data:/data
volumes:
  data:
```
