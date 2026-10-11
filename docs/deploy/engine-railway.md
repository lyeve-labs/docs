---
title: Engine on Railway
description: "Deploy the engine image to Railway and publish both of its APIs from one service."
sidebar:
  order: 7
---

Railway runs the engine image as a service built from a Docker image. One service can publish
both engine ports, each on its own domain. New accounts get trial credits, and after that a
service is billed by usage. A deployed service keeps running rather than sleeping, so check
Railway's current pricing before you leave it unattended.

:::tip[Best for]
- **Both APIs public from one service**, each on its own domain.
- **An engine that stays warm**, with a Railway volume for the signing key.
:::

## Before you start

- A Railway account and a project.
- A database. A Railway Postgres service in the same project works, or
  [Supabase](database-supabase.md) or [Neon](database-neon.md).

## Deploy

1. Add a service to the project from the Docker image
   `ghcr.io/lyeve-labs/lyeve-core:latest`. Pin a release tag in production. See
   [Docker images](docker.md#images-and-tags).
2. In the service's networking settings, give one public domain to port `3001` (the Admin API)
   and one to port `3002` (the Content API).
3. Add a volume mounted at `/var/lib/lyeve`, so the signing key survives a redeploy. The engine
   runs as uid `65532`, and Railway documents that an image running as a non-root user cannot
   write to an attached volume unless the service sets `RAILWAY_RUN_UID=0`, which runs the
   container as root. Set it, or the engine cannot write the key. In production the engine
   refuses to start when it cannot write the key.

Without the volume, every redeploy creates a new key and every token issued before it stops
working.

## Secrets

Set these in the service's variables:

```text
DATABASE_URL=postgres://<user>:<password>@<host>:5432/<dbname>?sslmode=require
JWT_SECRET=<openssl rand -hex 32>
ENCRYPTION_KEY=<a different openssl rand -hex 32>
LYEVE_AUDIT_HMAC_KEY=<a third openssl rand -hex 32>
RATE_LIMIT_RPS=100
SECURE_COOKIE=true
CORS_ORIGINS=https://your-app.example.com
RAILWAY_RUN_UID=0
```

Production refuses to start without these values.
[Configuration](../reference/configuration.md#what-production-refuses) lists every check.

## Database

For a Railway Postgres service, use the internal connection string Railway generates, which
stays on Railway's network. Use the public one only when something outside Railway reaches the
database. Production refuses the database user `postgres`, so create a dedicated user for the
engine. [Supported databases](index.md#supported-databases) lists the URL forms.

## Health checks

Set the service's health check path to `/readyz`. Both listeners serve `/healthz`, `/readyz`
and `/startup` without authentication. Do not use `/api/admin/health` or `/api/v1/health`:
they need authentication, so a check gets `401` and marks a healthy service down.
[Health endpoints](docker.md#health-endpoints) says what each one checks.

## Verify

```bash
curl https://<content-api-domain>/.well-known/jwks.json
curl https://<admin-api-domain>/api/admin/setup
```

The JWKS document is on the domain for port 3002, and the setup route on the domain for port
3001. A new install answers `{"setup_required":true,"token_source":"log"}`. The setup token is
the one-time `setup_token` in the service's log, or `LYEVE_SETUP_TOKEN` when you set it. The
[quickstart](../getting-started/quickstart.md) continues from step 2.

## The admin console

This page sets up the engine alone. The admin console runs as its own service, from the image
`ghcr.io/lyeve-labs/lyeve-admin`. [The admin image](docker.md#the-admin-image) lists its
settings and the routing it needs.

## Next

- [Engine on Fly.io](engine-fly.md): another host that publishes both APIs.
- [Production checklist](production-checklist.md): what to confirm before real traffic.
