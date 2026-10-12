---
title: Docker images
description: "Run the engine and admin console images on your own hosts: tags, ports, volumes, settings, TLS and health checks."
sidebar:
  order: 12
---

LyEve ships as two container images. Run them on any host that runs containers. This page
describes each image on its own. For a complete single-host stack with a database and a TLS
proxy, copy the files on [Docker Compose](docker-compose.md).

:::tip[Best for]
- **Trying the engine** on your machine in one command.
- **Your own servers or orchestrator**, when you want to know exactly what each image needs.
:::

## Images and tags

| Image | What it runs | Platforms |
|---|---|---|
| `ghcr.io/lyeve-labs/lyeve-core` | The engine: the Admin API, the Content API and every feature | `linux/amd64`, `linux/arm64` |
| `ghcr.io/lyeve-labs/lyeve-admin` | The admin console, a Node server that calls the engine for the browser | `linux/amd64`, `linux/arm64` |

Each release publishes three tags per image:

| Tag | Moves? |
|---|---|
| `0.50.4` | Never. One release, exactly |
| `0.50` | To the latest patch of that minor release |
| `latest` | To every new release |

Tags carry no `v` prefix: release `v0.50.4` is the image tag `0.50.4`. Pin an exact version in
production and upgrade on purpose.

## The engine image

| | |
|---|---|
| Ports | `3001` Admin API (`/api/admin`). `3002` Content API (`/api/v1`) and `/.well-known/jwks.json`. `3003` and `3004` are for the [gRPC API](../features/grpc-api.md) and stay closed unless you turn it on |
| User | uid `65532`, no shell in the image |
| Writable paths | `/var/lib/lyeve` for the signing key and the cached license. `/app/uploads` for media on local disk |
| Health check | Built in. `/app/lyeve healthcheck` asks `/healthz` on port `3002`, or on `PORT` when set |

The image sets `ADMIN_LISTEN_ADDR=0.0.0.0:3001`, `API_LISTEN_ADDR=0.0.0.0:3002` and
`STORAGE_LOCAL_PATH=/app/uploads`. Leave them unless you move a port.

Mount a volume at `/var/lib/lyeve`. The engine creates its Ed25519 signing key there on first
boot. Without the volume, a replaced container creates a new key and every token issued before
it stops working. A named Docker volume is writable by uid `65532`, because the image prepares
the directory with that owner.

Mount a second volume at `/app/uploads` if media stays on local disk, or send media to an
S3-compatible bucket. See [Object storage](../features/object-storage.md).

### Run it

```bash
docker run -d --name lyeve-engine \
  -e DATABASE_URL="postgres://lyeve:<password>@db.internal:5432/lyeve?sslmode=require" \
  -e JWT_SECRET="$(openssl rand -hex 32)" \
  -e ENCRYPTION_KEY="$(openssl rand -hex 32)" \
  -e LYEVE_AUDIT_HMAC_KEY="$(openssl rand -hex 32)" \
  -e RATE_LIMIT_RPS=100 \
  -e SECURE_COOKIE=true \
  -v lyeve-state:/var/lib/lyeve \
  -v lyeve-uploads:/app/uploads \
  -p 3001:3001 -p 3002:3002 \
  ghcr.io/lyeve-labs/lyeve-core:0.50.4
```

Generate the secrets once and store them. A new `ENCRYPTION_KEY` makes stored secrets
unreadable. `APP_ENV` defaults to `production`,
which refuses a weak configuration. [Configuration](../reference/configuration.md#what-production-refuses)
lists every check.

### Settings most deployments add

| Variable | Purpose |
|---|---|
| `LYEVE_CONSOLE_URL` | The admin console's public URL, `https` in production. Password reset and magic-link emails link to it. Unset in production, those features refuse to start and device sign-in answers `503` |
| `CORS_ORIGINS` | Browser origins allowed to call the APIs cross-origin, comma-separated |
| `TRUSTED_PROXIES` | CIDRs of your reverse proxies, so the engine reads the client address from `X-Forwarded-For`. Without it every request counts as the proxy's, and the per-address limits apply to all users together |
| `ADMIN_CONSOLE_KEY` | A secret of 32 characters or more, shared with the admin console, so sign-in limits count each browser |
| `LYEVE_SETUP_TOKEN` | The token the first-run setup asks for, 16 characters or more |
| `LYEVE_LICENSE_KEY` | Your license, for the paid features |

Email has its own settings on [Email](../features/email.md). Every variable, with its default,
is on [Configuration](../reference/configuration.md).

## The admin image

| | |
|---|---|
| Port | `3002` by default, set with `PORT` |
| User | `node` |
| Health check | Built in, an HTTP request to `/` on port `3002` |

| Variable | Purpose |
|---|---|
| `ORIGIN` | The public URL the browser uses, such as `https://admin.example.com`. Required, for the cross-site check on form posts and for cookies |
| `CORE_INTERNAL_URL` | Where the console's server reaches the engine. Every server-side `/api` call goes to this one base URL, so it must route `/api/admin` to the engine's port 3001 and the rest of `/api` to port 3002 |
| `CORE_API_INTERNAL_URL` | Where the API reference page sends the calls it runs under `/api/v1`. Set it to the same base as `CORE_INTERNAL_URL`, or to the engine's port 3002. The console refuses to start when it is unset or points at the console itself |
| `ADMIN_CONSOLE_KEY` | The same value the engine holds |
| `ADDRESS_HEADER`, `XFF_DEPTH` | Behind a proxy, `X-Forwarded-For` and `1`, so the console sees the browser's address |
| `PORT` | Listen port. Default `3002` |

A base URL such as `http://engine:3001` sends Content API calls to the wrong listener, and they
answer `404`. Put a small proxy in front of both engine ports and point the console at it, as
the [Compose file](docker-compose.md) does.

## Reverse proxy and TLS

Terminate TLS at a reverse proxy in front of both images. The browser loads the console and
also calls the engine on the console's origin, so the console's public site routes by path:

| Path | Goes to |
|---|---|
| `/api/admin/*` | engine `:3001` |
| `/api/*` and `/.well-known/*` | engine `:3002` |
| everything else | admin console `:3002` |

A Content API host for your sites and apps sends everything to engine `:3002`. Give the Admin
API no public host of its own unless a tool outside your network needs it.

With `SECURE_COOKIE=true` the engine redirects a plain-HTTP request to HTTPS with a `301` unless
it carries `X-Forwarded-Proto: https`. A proxy that terminates TLS sets that header. A
plain-HTTP hop inside your network, such as the console's own calls, has to set it too. The
[Caddyfile](docker-compose.md#the-compose-file) shows both.

## Health endpoints

Both engine listeners serve three probes at the root, with no authentication and over plain
HTTP even with `SECURE_COOKIE=true`. Each answers JSON listing the checks it ran.

| Path | Use | Answers `503` when |
|---|---|---|
| `/healthz` | Liveness | The database does not answer a ping. A restart can clear a stuck connection pool |
| `/readyz` | Readiness | The engine is still starting or is shutting down, the database does not answer, or less than 100 MiB is free on the uploads path |
| `/startup` | Startup | The checks have not all passed once yet. After that it always answers `200` |

Point every orchestrator probe and uptime monitor at these. `/api/admin/health` and
`/api/v1/health` also exist, but they need authentication, so a prober gets `401` and marks a
healthy engine down. The thresholds are on
[Configuration](../reference/configuration.md#health-checks-and-shutdown).

## Verify

```bash
curl -s http://localhost:3001/readyz
curl -s -H 'X-Forwarded-Proto: https' http://localhost:3002/.well-known/jwks.json
curl -s -H 'X-Forwarded-Proto: https' http://localhost:3001/api/admin/setup
```

The probe answers `{"status":"ok", ...}` with its checks. The other two paths redirect to
HTTPS under `SECURE_COOKIE=true`, so these local checks send the header your proxy would. The
JWKS response lists the engine's public signing key. A new install answers the setup request
with `{"setup_required":true,"token_source":"log"}`: the token is the one the engine printed,
which `docker logs lyeve-engine 2>&1 | grep setup_token` finds. With `LYEVE_SETUP_TOKEN` set,
`token_source` is `env`. The [quickstart](../getting-started/quickstart.md) continues from step
2.

## Next

- [Docker Compose](docker-compose.md): the whole stack in two files.
- [Kubernetes](kubernetes.md): the same images on a cluster.
- [Production checklist](production-checklist.md): what to confirm before go-live.
