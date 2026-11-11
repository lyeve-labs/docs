---
title: Docker images
description: "Run the engine and LyEve Console images on your own hosts: tags, ports, volumes, settings, TLS and health checks."
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
| `ghcr.io/lyeve-labs/lyeve-console` | LyEve Console, a Node server that calls the engine for the browser | `linux/amd64`, `linux/arm64` |

Each release publishes three tags per image:

| Tag | Moves? |
|---|---|
| `0.58.0` | Never. One release, exactly |
| `0.58` | To the latest patch of that minor release |
| `latest` | To every new release |

Tags carry no `v` prefix: release `v0.58.0` is the image tag `0.58.0`. Pin an exact version in
production and upgrade on purpose.

### Release candidates

Before a release, the engine and the console publish one or more release candidates,
tagged `X.Y.Z-rc.N`. A candidate is the build we intend to ship, published so you can try the
upgrade on a copy of your own deployment before the release is final. An urgent fix can ship
without one.

| Tag | Moves? |
|---|---|
| `X.Y.Z-rc.1`, `X.Y.Z-rc.2`, ... | Never. One candidate, exactly |

A candidate is published under its exact tag and nothing else. It never moves `latest` or the
minor tag, so you only run one by naming it:

```bash
docker pull ghcr.io/lyeve-labs/lyeve-core:X.Y.Z-rc.N
```

What a candidate promises:

- The release `X.Y.Z` is the last candidate, with no other change.
- A database that ran a candidate upgrades to `X.Y.Z` and to every later release.
- There is no downgrade from a candidate to an earlier release. Run candidates on a copy of your
  data, never on the only one.

Report a problem with a candidate as you would with a release. A fix produces the next candidate.

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
the directory with that owner. The volume keeps the key, not the sessions: refresh tokens live
in the engine's memory unless `CACHE_DRIVER=redis` and `REDIS_URL` point at a Redis, so a new container still signs
everyone out once their access token expires, 15 minutes by default.

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
  --memory 1g \
  -v lyeve-state:/var/lib/lyeve \
  -v lyeve-uploads:/app/uploads \
  -p 3001:3001 -p 3002:3002 \
  ghcr.io/lyeve-labs/lyeve-core:0.62.0
```

`--memory` caps the engine's memory, and the engine sets Go's soft memory limit to 85% of it, so
it collects garbage harder as it nears the cap instead of being killed at it.

Generate the secrets once and store them. A new `ENCRYPTION_KEY` makes stored secrets
unreadable. `APP_ENV` defaults to `production`,
which refuses a weak configuration. [Configuration](../reference/configuration.md#what-production-refuses)
lists every check.

### Settings most deployments add

| Variable | Purpose |
|---|---|
| `LYEVE_CONSOLE_URL` | The console's public URL, `https` in production. Password reset and magic-link emails link to it. Unset in production, those features refuse to start and device sign-in answers `503` |
| `CORS_ORIGINS` | Browser origins allowed to call the APIs cross-origin, comma-separated |
| `TRUSTED_PROXIES` | CIDRs of your reverse proxies, so the engine reads the client address from `X-Forwarded-For`. Without it every request counts as the proxy's, and the per-address limits apply to all users together |
| `LYEVE_CONSOLE_KEY` | A secret of 32 characters or more, shared with the console, so sign-in limits count each browser |
| `LYEVE_SETUP_TOKEN` | The token the first-run setup asks for, 16 characters or more |
| `LYEVE_LICENSE_KEY` | Your license, for the paid features |

Email has its own settings on [Email](../features/email.md). Every variable, with its default,
is on [Configuration](../reference/configuration.md).

## The admin image

| | |
|---|---|
| Port | `3002` by default, set with `PORT` |
| User | `node` |
| Health check | Built in, an HTTP request to `/readyz` on port `3002`. It answers `503` while the console cannot reach the engine |

| Variable | Purpose |
|---|---|
| `ORIGIN` | The public URL the browser uses, such as `https://admin.example.com`. Required, for the cross-site check on form posts and for cookies |
| `CORE_INTERNAL_URL` | Where the console's server reaches the engine. Every server-side `/api` call goes to this one base URL, so it must route `/api/admin` to the engine's port 3001 and the rest of `/api` to port 3002 |
| `CORE_API_INTERNAL_URL` | Where the API reference page sends the calls it runs under `/api/v1`. Set it to the same base as `CORE_INTERNAL_URL`, or to the engine's port 3002. The console refuses to start when it is unset or points at the console itself |
| `LYEVE_CONSOLE_KEY` | The same value the engine holds |
| `ADDRESS_HEADER`, `XFF_DEPTH` | Behind a proxy, `X-Forwarded-For` and the number of proxies, usually `1`, so the console sees the browser's address. With an `https` `ORIGIN` the console refuses to start without `ADDRESS_HEADER` |
| `SIGN_IN_ORGANIZATION` | `true` shows **Organization (optional)** on the sign-in, password reset and email link pages, for a console that serves more than one organization on an address that names none. Off by default |
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
| everything else | console `:3002` |

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
with `{"setup_required":true,"token_source":"log"}`: the token is the one the engine wrote to its standard error,
which `docker logs lyeve-engine 2>&1 | grep setup_token` finds. With `LYEVE_SETUP_TOKEN` set,
`token_source` is `env`. The [quickstart](../getting-started/quickstart.md) continues from step
2.

## Upgrading to 0.62.0 and console 0.28.0

Upgrade the engine before the console. Neither release adds a setting or refuses one that
0.61.0 and 0.27.0 accept, and the engine applies its new migrations on its first start. The
builder now starts on every install, free with 2 sites per tenant, 10 pages in each and 20 in
all, and a `LYEVE_PLUGINS` that names features has to include `builder` to start it. With a
license that includes `ai`, every signed-in role sees the assistant in the console's header.
`GET /api/admin/openapi.json` answers every signed-in role with the operations that role may
call. Under a CPU limit of one core or less, the engine runs its scheduler on one thread unless
`GOMAXPROCS` or `CPU_QUOTA` says otherwise. Engine 0.62.0 is built with Go 1.27.4, which fixes
a crash a client could cause in its HTTP server. As with 0.61.0, both releases still read
`ADMIN_CONSOLE_KEY`, and a later minor release stops, so rename it to `LYEVE_CONSOLE_KEY` if
you have not. Console 0.28.0 is also published under `ghcr.io/lyeve-labs/lyeve-admin`, but run
`ghcr.io/lyeve-labs/lyeve-console`. An install on 0.60.2 also reads the 0.61.0 changes, which
still apply: content entries without `schema_name`, list filters on numbers and relations that
answer `400`, and messages that say content type where they said schema. An install on a
release before 0.60.2 reads the next section first.
[Docker Compose](docker-compose.md#upgrading-to-0620-and-console-0280) explains each step.

## Upgrading to 0.60.2 and console 0.26.0

Engine 0.60.0 and 0.60.1 have no published image, so run 0.60.2. Upgrade the engine before the
console. Engine 0.60.0 refuses to start with an `APP_ENV` other than `production` or
`development`, a `METRICS_TOKEN` shorter than 32 characters, or a `JWT_SECRETS` entry shorter
than 16, and with a placeholder in either secret. Rename `ADMIN_CONSOLE_KEY` to
`LYEVE_CONSOLE_KEY` on both images, and `ADMIN_CONSOLE_KEY_PREVIOUS` to
`LYEVE_CONSOLE_KEY_PREVIOUS` on the engine. The old names work for one more minor release. A
console with an `https` `ORIGIN` refuses to start without `ADDRESS_HEADER`. A Loki or
Elasticsearch log sink on your own network needs that network in
`LOGGING_ALLOWED_PRIVATE_NETWORKS`.
[Docker Compose](docker-compose.md#upgrading-to-0602-and-console-0260) explains each step, a
SAML change in 0.60.2, and two changes your clients see: cached reads made with a credential
and a realtime key's content scope.

## Upgrading to console 0.25.0

Console 0.25.0 is published as `ghcr.io/lyeve-labs/lyeve-console`, so run that image from now
on. Every 0.25 release is also published under `ghcr.io/lyeve-labs/lyeve-admin` with the same
tags and digests. The old name has received every release through 0.28.0 so far, and the tags
already published there stay.
[Docker Compose](docker-compose.md#upgrading-to-console-0250) has the details.

## Upgrading to 0.58.0 and console 0.24.0

Upgrade the engine before the console, and back up the database first. A new install's setup
token is on the engine's standard error, outside the log. With implicit TLS off, a send to a
mail relay that does not offer STARTTLS fails, so fix the relay before you upgrade. A quota's
grace period now delays its block. Console 0.24.0 asks for an organization at sign-in only with
`SIGN_IN_ORGANIZATION=true`.
[Docker Compose](docker-compose.md#upgrading-to-0580-and-console-0240) explains each step and
the changes your API clients and your directory see.

## Upgrading to 0.57.0 and console 0.23.0

Upgrade the engine before the console, and back up the database first. On MySQL or SQL Server,
stop every engine container before you start 0.57.0: its first start rebuilds the accounts
table, which took about 18 seconds per million accounts on MySQL and about 3 seconds per
million on SQL Server on a test host, and an older engine must not run beside it. Content
events about entries saved in the console now carry their fields at the top level of `data`.
[Docker Compose](docker-compose.md#upgrading-to-0570-and-console-0230) explains each step and
the changes your API clients, webhook receivers and flows see.

## Upgrading to 0.56.0 and console 0.22.0

Upgrade the engine before the console: console 0.22.0 reads content history from engine 0.56.0.
Back up the database first, because an engine before 0.56.0 refuses to start on a database
0.56.0 has upgraded. A tenant that already holds content keeps content history on for every
content type. [Docker Compose](docker-compose.md#upgrading-to-0560-and-console-0220) explains
each step and the changes your API clients see.

## Upgrading to 0.55.0 and console 0.21.0

Upgrade the engine before the console: console 0.21.0 needs engine 0.55.0 or later. Stop every
engine container before you start 0.55.0, because an older engine beside it breaks the audit
hash chain. Review your retention policies first, because the first boot enforces them. A tenant
with an active legal hold can no longer be deleted and answers `409`. A mail relay on loopback or
a private network needs every address its name resolves to in `SMTP_ALLOWED_PRIVATE_NETWORKS`,
both `127.0.0.1` and `::1` for `localhost`. Run one console container, or give your proxy cookie
affinity when you run several. [Docker Compose](docker-compose.md#upgrading-to-0550-and-console-0210)
explains each step and the changes your API clients see.

## Next

- [Docker Compose](docker-compose.md): the whole stack in two files.
- [Kubernetes](kubernetes.md): the same images on a cluster.
- [Production checklist](production-checklist.md): what to confirm before go-live.
