---
title: Free-tier stack
description: "Run the engine and its database for $0 a month on free hosting plans, and know the limits before you rely on it."
sidebar:
  order: 2
---

This recipe runs the engine and a Postgres database at $0 a month: one free account for the
engine, one for the database, wired together with a handful of environment variables. The
result is a headless engine over HTTPS, managed with `curl` or your own tooling.
It suits a demo, a staging environment or a low-traffic project. To compare it with the other
ways to run LyEve, see [Choose where to run LyEve](index.md).

## Before you start

- An account on an engine host with a free plan: [Render](engine-render.md) or
  [Google Cloud Run](engine-cloud-run.md). Each publishes one of the two APIs. [Railway](engine-railway.md)
  and [Fly.io](engine-fly.md) publish both, but they bill by usage after a trial.
- An account on [Supabase](database-supabase.md) or [Neon](database-neon.md) for the database.
- `openssl` and `curl` on your machine.

## Steps

1. **Create the database.** Make a free Postgres project and copy the connection string its
   page recommends: the session pooler on [Supabase](database-supabase.md), the direct string
   on [Neon](database-neon.md).

2. **Generate the secrets.** Run this three times and keep each value somewhere durable:

   ```bash
   openssl rand -hex 32
   ```

3. **Deploy the engine image** `ghcr.io/lyeve-labs/lyeve-core` on your engine host, following
   its page, with these variables:

   ```text
   DATABASE_URL=<the connection string from step 1>
   DATABASE_MAX_CONNECTIONS=5
   JWT_SECRET=<first secret>
   ENCRYPTION_KEY=<second secret>
   LYEVE_AUDIT_HMAC_KEY=<third secret>
   RATE_LIMIT_RPS=100
   SECURE_COOKIE=true
   CORS_ORIGINS=https://your-app.example.com
   ```

   `APP_ENV` defaults to `production`, which refuses to start without the secrets,
   `RATE_LIMIT_RPS` and `SECURE_COOKIE=true`. [Configuration](../reference/configuration.md#what-production-refuses)
   lists every check. `CORS_ORIGINS` names the browser apps that will call the Content API.

4. **Check the engine.** Ask the Content API for the engine's public key, or ask the Admin API
   whether setup is needed, whichever your host publishes:

   ```bash
   curl https://<content-api-url>/.well-known/jwks.json
   curl https://<admin-api-url>/api/admin/setup
   ```

   The Admin API answers `{"setup_required":true,"token_source":"log"}`. The setup token is the
   one-time `setup_token` the engine printed to its log at startup.

5. **Create the first administrator.** Follow the [quickstart](../getting-started/quickstart.md)
   from step 2, against your Admin API URL. On a host that publishes one API, publish port
   `3001` for this step.

## Limits of the free stack

- **Cold starts.** Both free engine hosts stop the container after a period with no traffic:
  Render free web services, and Cloud Run with `--min-instances=0`. The first request after that
  waits for three things: the host starting the container, the database resuming if it slept
  (Neon does), and the engine's own start. The engine logs its share as `boot_ms` on its
  `serving` line, so the log shows how much of a slow first request was the engine.
- **Signing out on restart.** The engine keeps its signing key at `JWT_KEY_PATH`
  (`/var/lib/lyeve/jwt_key.json`) and creates a new one when the file is missing. On a host with
  no persistent disk, every cold start or redeploy makes a new key, and every token issued
  before it stops working. Where the host offers a volume, mount it at `/var/lib/lyeve`, writable
  by uid `65532`.
- **One engine service.** Two services started from the image each create their own key, so a
  token from one is refused by the other. Do not run a second service to publish the other port.
- **Connection limits.** Free Postgres plans cap connections. Keep `DATABASE_MAX_CONNECTIONS`
  small, because a redeploy can briefly run the old and new engine side by side.
- **No disk for media.** Uploads stored on local disk, the default, are lost at a cold start or
  a redeploy. Store them in an S3-compatible bucket. See
  [Object storage](../features/object-storage.md).
- **No admin console.** The console needs a host that runs a second container behind a proxy.
  See [The admin console needs a server](index.md#the-admin-console-needs-a-server-not-vercel-or-cloudflare-pages).
- **No SQLite.** SQLite, Turso and libSQL cannot replace the database. See
  [supported databases](index.md#supported-databases).

## Next

- [Production checklist](production-checklist.md) before real data goes behind it.
- [Docker Compose](docker-compose.md) to add the admin console on a server you control.
