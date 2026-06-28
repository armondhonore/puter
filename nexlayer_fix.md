# Nexlayer fix (authoritative, pinned)

Root cause of the in-browser ERR_SSL_PROTOCOL_ERROR (curl only saw a 302):
puter's backend used Express's default `subdomain offset` of 2. The 4-label
deploy host `relaxed-weasel-puter.cloud.nexlayer.ai` was therefore misread as a
user-site subdomain, and `/` 302-redirected to
`site.relaxed-weasel-puter.cloud.nexlayer.ai` — a subdomain with NO Nexlayer TLS
cert, so the browser could not complete the TLS handshake.

Fix is in SOURCE on this branch (must be built, not run from the published
image):
- `src/backend/server.ts` — derive `subdomain offset` from `config.domain`'s
  label count, so the apex correctly reports zero subdomains and the desktop GUI
  root route serves `/` directly on the same (cert-valid) host.
- `src/backend/core/http/middleware/hostRedirects.ts` — also short-circuit when
  the request host equals the apex GUI domain.

Build from the Dockerfile below (do NOT substitute the published
`ghcr.io/heyputer/puter:latest` image — it lacks this patch). The nexlayer.yaml
keeps `image: "# filled by pipeline"` so the pipeline injects the freshly built
image. `trust_proxy: 1` makes `req.protocol` resolve to https behind the
Nexlayer edge (otherwise puter builds redirect URLs with http and loops).

## Fixed Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.7
# OSS Puter image, built from source so the subdomain-redirect fix compiles in.

# ---- Build stage ----
FROM node:24-slim AS build
ENV HTTP_PORT=80
ENV MARIADB_DATABASE=puter
ENV MARIADB_PASSWORD=replace-with-strong-password
ENV MARIADB_ROOT_PASSWORD=replace-with-strong-password
ENV MARIADB_USER=puter
ENV S3_ACCESS_KEY=puter
ENV S3_BUCKET=puter-local
ENV S3_SECRET_KEY=replace-with-strong-secret

WORKDIR /opt/puter

# Build toolchain needed for native deps (bcrypt, sharp, better-sqlite3, …).
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 make g++ git && \
    rm -rf /var/lib/apt/lists/*

ENV HUSKY=0
ENV npm_config_fund=false
ENV npm_config_audit=false

# Dependency layer — manifests first so the npm-install layer stays cached.
COPY package.json package-lock.json ./
COPY src/backend/package.json src/backend/
COPY src/gui/package.json src/gui/
COPY src/puter-js/package.json src/puter-js/package-lock.json src/puter-js/
COPY src/worker/package.json src/worker/
COPY src/docs/package.json src/docs/
COPY tools/extensionSetup.mjs tools/extensionSetup.mjs

RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Source layer.
COPY . .

RUN npm run build:ts
RUN set -e; \
    (cd src/gui && node ./build.js) & gui_pid=$!; \
    (cd src/puter-js && npm run build) & pjs_pid=$!; \
    wait $gui_pid; \
    wait $pjs_pid

# ---- Runtime stage ----
FROM node:24-slim
WORKDIR /opt/puter
RUN apt-get update && \
    apt-get install -y --no-install-recommends git wget && \
    rm -rf /var/lib/apt/lists/*
COPY --from=build --chown=node:node /opt/puter .
RUN mkdir -p /etc/puter /var/puter && \
    chown -R node:node /etc/puter /var/puter
ENV PUTER_CONFIG_PATH=/etc/puter/config.json
ENV NODE_OPTIONS=--enable-source-maps
EXPOSE 4100
USER node
CMD ["node", "./dist/src/backend/index.js"]
```

## Fixed nexlayer.yaml

```yaml
application:
  name: puter
  pods:
  - name: app
    image: "# filled by pipeline"
    path: /
    # Boot: write runtime config to /tmp/puter (node-writable; /opt/puter is
    # root-owned). sqlite + local S3 (fauxqs) under /tmp/puter, DynamoDB
    # in-memory — ephemeral, fine for this test deploy. Config sets
    # domain/protocol=https, trust_proxy=1 (so req.protocol is https behind the
    # edge), and hosting domains as distinct sub-subdomains (never the apex), so
    # the source subdomain-offset fix serves the desktop GUI on the apex
    # directly with no cross-host / protocol-downgrade redirect.
    command: sh -c "mkdir -p /tmp/puter/runtime && echo eyJkb21haW4iOiAicmVsYXhlZC13ZWFzZWwtcHV0ZXIuY2xvdWQubmV4bGF5ZXIuYWkiLCAicHJvdG9jb2wiOiAiaHR0cHMiLCAidHJ1c3RfcHJveHkiOiAxLCAiYWxsb3dfYWxsX2hvc3RfdmFsdWVzIjogdHJ1ZSwgInN0YXRpY19ob3N0aW5nX2RvbWFpbiI6ICJzaXRlLnJlbGF4ZWQtd2Vhc2VsLXB1dGVyLmNsb3VkLm5leGxheWVyLmFpIiwgInN0YXRpY19ob3N0aW5nX2RvbWFpbl9hbHQiOiAiaG9zdC5yZWxheGVkLXdlYXNlbC1wdXRlci5jbG91ZC5uZXhsYXllci5haSIsICJwcml2YXRlX2FwcF9ob3N0aW5nX2RvbWFpbiI6ICJhcHAucmVsYXhlZC13ZWFzZWwtcHV0ZXIuY2xvdWQubmV4bGF5ZXIuYWkiLCAiZGF0YWJhc2UiOiB7ImVuZ2luZSI6ICJzcWxpdGUiLCAicGF0aCI6ICIvdG1wL3B1dGVyL3J1bnRpbWUvcHV0ZXItZGF0YWJhc2Uuc3FsaXRlIn0sICJzMyI6IHsibG9jYWxDb25maWciOiB7ImRhdGFEaXIiOiAiL3RtcC9wdXRlci9ydW50aW1lL2ZhdXhxcy1kYXRhIiwgInMzU3RvcmFnZURpciI6ICIvdG1wL3B1dGVyL3J1bnRpbWUvZmF1eHFzLXMzLWRhdGEifX0sICJkeW5hbW8iOiB7ImluTWVtb3J5IjogdHJ1ZX19 | base64 -d > /tmp/puter/config.json && export PUTER_CONFIG_PATH=/tmp/puter/config.json && exec node ./dist/src/backend/index.js"
    servicePorts:
    - 4100
    vars:
      PUTER_DEFAULT_USER: admin
      PUTER_DEFAULT_USER_PASSWORD: nexlayer2024
```
