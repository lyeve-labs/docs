---
title: Engine on Google Cloud Run
description: "Deploy the engine image to Google Cloud Run, publish one of its two APIs, and keep its secrets in Secret Manager."
sidebar:
  order: 8
---

Cloud Run runs the engine image as a request-driven service that can scale to zero. Its free
monthly allotment of requests, CPU and memory covers a low-traffic engine.

:::tip[Best for]
- **A staging or demo engine** that costs nothing while nobody uses it.
- **An engine inside Google Cloud**, next to a Cloud SQL database.
:::

:::caution[What you give up]
- A Cloud Run service publishes one port, so it serves the Admin API or the Content API, not
  both. For both, use [Railway](engine-railway.md) or [Fly.io](engine-fly.md).
- An instance has no persistent disk, so every new instance creates its own signing key. After
  a scale to zero, users sign in again. Refresh tokens also live in the engine's memory unless
  `CACHE_DRIVER=redis` and `REDIS_URL` point at a Redis. Without Redis, a new instance cannot
  renew a session, so users sign in again.
:::

## Before you start

- A Google Cloud project with Cloud Run, Artifact Registry and Secret Manager turned on, and the
  `gcloud` CLI signed in to it.
- A Postgres database: Cloud SQL, or [Supabase](database-supabase.md) or
  [Neon](database-neon.md) for a free one.
- An Artifact Registry remote repository whose upstream is `https://ghcr.io`. Cloud Run deploys
  images from Artifact Registry or Docker Hub, not from `ghcr.io` directly.

## Deploy

Choose the port. `--port=3001` publishes the Admin API, which you need to create the first
administrator and define content types. `--port=3002` publishes the Content API for your sites
and apps.

```bash
gcloud run deploy lyeve-engine \
  --image=<region>-docker.pkg.dev/<project>/<remote-repo>/lyeve-labs/lyeve-core:latest \
  --port=3001 \
  --allow-unauthenticated \
  --min-instances=0 \
  --max-instances=1 \
  --set-env-vars="SECURE_COOKIE=true,RATE_LIMIT_RPS=100,DATABASE_MAX_CONNECTIONS=5" \
  --set-secrets="DATABASE_URL=lyeve-database-url:latest,JWT_SECRET=lyeve-jwt-secret:latest,ENCRYPTION_KEY=lyeve-encryption-key:latest,LYEVE_AUDIT_HMAC_KEY=lyeve-audit-hmac-key:latest"
```

Keep `--max-instances=1`, and do not deploy a second service from the image for the other port.
Each instance signs with a key of its own, so a token from one is refused by another.
With the default `--min-instances=0`, the first request after a scale to zero waits for Cloud
Run to start an instance, then for the engine to start. The engine logs its share as `boot_ms`
on its `serving` line. `--min-instances=1` removes cold starts but keeps an instance running,
which is billed.

Pin a release tag instead of `latest` in production. See
[Docker images](docker.md#images-and-tags).

## Secrets

Store the four secrets in Secret Manager and pass them with `--set-secrets`, as above, so they
stay out of the service's visible configuration:

```bash
printf %s "$(openssl rand -hex 32)" | gcloud secrets create lyeve-jwt-secret --data-file=-
printf %s "$(openssl rand -hex 32)" | gcloud secrets create lyeve-encryption-key --data-file=-
printf %s "$(openssl rand -hex 32)" | gcloud secrets create lyeve-audit-hmac-key --data-file=-
printf %s "<your connection string>" | gcloud secrets create lyeve-database-url --data-file=-
```

The service's account needs permission to read them. Production refuses to start without these
secrets, `SECURE_COOKIE=true` and a non-zero `RATE_LIMIT_RPS`.
[Configuration](../reference/configuration.md#what-production-refuses) lists every check.

## Database

- **Cloud SQL for PostgreSQL** keeps the database in the same cloud. Connect it with
  `--add-cloudsql-instances` or the Cloud SQL Auth Proxy. Cloud SQL is billed separately.
- **Supabase or Neon** connect over the internet. Both have a free plan, which makes them the
  path for a $0 stack. Each page says which connection string to use.

Keep `DATABASE_MAX_CONNECTIONS` small on a free database plan.

## Health checks

By default Cloud Run checks that the container accepts connections on `--port`. For HTTP checks,
set a startup probe on `/startup` and a liveness probe on `/healthz`. Both listeners serve
`/healthz`, `/readyz` and `/startup` without authentication and over plain HTTP. Do not use
`/api/admin/health` or `/api/v1/health`: they need authentication, so a probe gets `401` and
marks a healthy instance down. [Health endpoints](docker.md#health-endpoints) says what each
one checks.

## Verify

With `--port=3001`:

```bash
curl https://lyeve-engine-<hash>-<region>.a.run.app/api/admin/setup
```

A new install answers `{"setup_required":true,"token_source":"log"}`. The setup token is the
one-time `setup_token` in the service's log, or `LYEVE_SETUP_TOKEN` when you set it. The
[quickstart](../getting-started/quickstart.md) continues from step 2.

With `--port=3002`, fetch `/.well-known/jwks.json` instead. Only the Content API serves it.

## Next

- [Free-tier stack](free-tier-overview.md): the $0 recipe and its limits.
- [Run the engine and the console together](../getting-started/engine-and-console.md): the admin
  console in front of the engine, on one address.
- [Production checklist](production-checklist.md): what to confirm before real traffic.
