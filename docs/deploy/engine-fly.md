---
title: Engine on Fly.io
description: "Deploy the engine image to Fly.io with both APIs published over TLS from one Machine."
sidebar:
  order: 5
---

Fly.io runs the engine image as a long-lived container, a Machine. Fly bills by usage, and new
accounts get a short trial rather than a free plan. A Machine with `auto_stop_machines` stops
when idle and starts on the next request.

:::tip[Best for]
- **Both APIs public from one service**, the Content API on port 443 and the Admin API on 8443.
- **A small engine close to your users**, in the region you pick.
:::

:::caution[What you give up]
- The Admin API on port 8443 answers only on the app's IPv6 address, or on a dedicated IPv4
  address, which Fly bills for.
- Fly gives a Machine a blank file system each time it starts. The engine then creates a new
  signing key, so every start, an automatic stop and start included, signs everyone out. A Fly
  volume mounted at `/var/lib/lyeve` keeps the key, provided uid `65532` can write to it. In
  production the engine refuses to start when it cannot write the key.
:::

## Before you start

- A Fly.io account and the `fly` CLI signed in.
- A Postgres, MySQL or SQL Server database the Machine can reach. On
  [Supabase](database-supabase.md) or [Neon](database-neon.md), use the connection string each
  page recommends.

## Deploy

Save this as `fly.toml`:

```toml
app = "your-lyeve-engine"
primary_region = "iad"

[build]
  image = "ghcr.io/lyeve-labs/lyeve-core:latest"

# Content API: /api/v1 and /.well-known/jwks.json
[[services]]
  internal_port = 3002
  protocol = "tcp"

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]

  [[services.http_checks]]
    interval = "15s"
    timeout = "2s"
    grace_period = "20s"
    method = "get"
    path = "/readyz"

# Admin API: /api/admin
[[services]]
  internal_port = 3001
  protocol = "tcp"

  [[services.ports]]
    port = 8443
    handlers = ["tls", "http"]

  [[services.http_checks]]
    interval = "15s"
    timeout = "2s"
    grace_period = "20s"
    method = "get"
    path = "/readyz"
```

The engine listens on `3001` for the Admin API and `3002` for the Content API, and the image
already binds both on all interfaces. Two services cannot share external port 443 in one app,
so the Content API takes 443 and the Admin API takes 8443. If nothing outside Fly needs the
Admin API, drop its `[[services]]` block and reach port 3001 over Fly's private network.

Run one Machine. A second Machine signs with a key of its own and refuses the first one's
tokens. Pin a release tag instead of `latest` in production. See
[Docker images](docker.md#images-and-tags).

## Secrets

Set every setting with `fly secrets set` rather than in `[env]`:

```bash
fly secrets set \
  DATABASE_URL="postgres://<user>:<password>@<host>:5432/<dbname>?sslmode=require" \
  JWT_SECRET="$(openssl rand -hex 32)" \
  ENCRYPTION_KEY="$(openssl rand -hex 32)" \
  LYEVE_AUDIT_HMAC_KEY="$(openssl rand -hex 32)" \
  SECURE_COOKIE=true \
  RATE_LIMIT_RPS=100 \
  CORS_ORIGINS="https://your-app.example.com"
```

Generate `ENCRYPTION_KEY` once and keep it. A new value makes stored secrets, such as MFA
seeds, unreadable. Production refuses to start without these values.
[Configuration](../reference/configuration.md#what-production-refuses) lists every check, and
every other variable. Add `LYEVE_LICENSE_KEY` for paid features.

## Database

Point `DATABASE_URL` at your database. The engine reads the type from the scheme, as
[supported databases](index.md#supported-databases) shows. Use `sslmode=require` for a
PostgreSQL database outside Fly's private network, or the engine logs a warning on every start.

## Health checks

The checks in `fly.toml` use `/readyz`. Both listeners serve `/healthz`, `/readyz` and
`/startup` without authentication, and over plain HTTP even with `SECURE_COOKIE=true`. Do not
use `/api/admin/health` or `/api/v1/health`: they need authentication, so a check gets `401` and
marks a healthy Machine down. [Health endpoints](docker.md#health-endpoints) says what each one
checks.

## Verify

```bash
fly deploy
fly status
curl https://your-lyeve-engine.fly.dev/.well-known/jwks.json
curl -6 https://your-lyeve-engine.fly.dev:8443/api/admin/setup
```

The first `curl` lists the engine's public signing key. The second goes over IPv6, because port
8443 does not answer on a shared IPv4 address, and a new install answers
`{"setup_required":true,"token_source":"log"}`. The setup token is the one-time `setup_token` in
`fly logs`, or `LYEVE_SETUP_TOKEN` when you set it. The
[quickstart](../getting-started/quickstart.md) continues from step 2.

## Next

- [Free-tier stack](free-tier-overview.md): the $0 recipe and its limits.
- [Production checklist](production-checklist.md): what to confirm before real traffic.
