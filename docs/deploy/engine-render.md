---
title: Engine on Render
description: "Deploy the engine image to a Render web service and publish one of its two APIs."
sidebar:
  order: 6
---

Render runs the engine image as a web service. The free instance type works for a demo or a
staging engine.

:::tip[Best for]
- **A free engine for evaluation, staging or a low-traffic internal tool.**
:::

:::caution[What you give up]
- Render routes one public port per service, so the engine publishes the Admin API or the
  Content API, not both. For both, use [Railway](engine-railway.md) or [Fly.io](engine-fly.md).
- A free web service spins down after a period with no traffic. The next request waits for
  Render to start the container, then for the engine to start and connect to the database.
- A free web service has no persistent disk, so every start creates a new signing key and users
  sign in again. Refresh tokens also live in the engine's memory unless `CACHE_DRIVER=redis` and
  `REDIS_URL` point at a Redis. Without Redis, a new instance cannot renew a session, so users
  sign in again.
:::

## Before you start

- A Render account.
- A Postgres database. On [Supabase](database-supabase.md) or [Neon](database-neon.md), use the
  connection string each page recommends.

## Deploy

1. Create a web service that deploys an existing image, and enter
   `ghcr.io/lyeve-labs/lyeve-core:latest`. Pin a release tag in production. See
   [Docker images](docker.md#images-and-tags).
2. Choose the API with `PORT`. Render sends traffic to the port `PORT` names, and the image's
   built-in health check probes that port too. `PORT=3001` publishes the Admin API, which you
   need to create the first administrator and define content types. `PORT=3002` publishes the
   Content API for your sites and apps.

Do not run a second Render service from the image for the other port. Each service creates its
own signing key, so a token issued by one is refused by the other.

## Secrets

Set these in the service's environment:

```text
DATABASE_URL=postgres://<user>:<password>@<host>:5432/<dbname>?sslmode=require
DATABASE_MAX_CONNECTIONS=5
JWT_SECRET=<openssl rand -hex 32>
ENCRYPTION_KEY=<a different openssl rand -hex 32>
LYEVE_AUDIT_HMAC_KEY=<a third openssl rand -hex 32>
RATE_LIMIT_RPS=100
SECURE_COOKIE=true
CORS_ORIGINS=https://your-app.example.com
PORT=3001
```

Production refuses to start without these values.
[Configuration](../reference/configuration.md#what-production-refuses) lists every check.

## Database

Free Postgres plans have tight connection limits, and an engine that restarts after every
spin-down reconnects each time, so keep `DATABASE_MAX_CONNECTIONS` small.
[Supported databases](index.md#supported-databases) lists the URL forms.

## Health checks

Set the service's health check path to `/readyz`. Both listeners serve `/healthz`, `/readyz`
and `/startup` without authentication. Do not use `/api/admin/health` or `/api/v1/health`:
they need authentication, so a check gets `401` and marks a healthy service down.
[Health endpoints](docker.md#health-endpoints) says what each one checks.

## Verify

With `PORT=3001`:

```bash
curl https://your-service.onrender.com/api/admin/setup
```

A new install answers `{"setup_required":true,"token_source":"log"}`. The setup token is the
one-time `setup_token` in the service's log, or `LYEVE_SETUP_TOKEN` when you set it. The
[quickstart](../getting-started/quickstart.md) continues from step 2.

With `PORT=3002`:

```bash
curl https://your-service.onrender.com/.well-known/jwks.json
```

The first request after idle time is slow, and the next ones are fast until the service spins
down again. Part of that wait is Render starting the container and part is the engine's own
start, which the engine logs as `boot_ms` on its `serving` line.

## The admin console

This page sets up the engine alone. The admin console runs as its own service, from the image
`ghcr.io/lyeve-labs/lyeve-admin`. [The admin image](docker.md#the-admin-image) lists its
settings and the routing it needs.
[Run the engine and the console together](../getting-started/engine-and-console.md) shows the
two side by side on one address.

## Next

- [Free-tier stack](free-tier-overview.md): the $0 recipe and its limits.
- [Production checklist](production-checklist.md): what to confirm before real traffic.
