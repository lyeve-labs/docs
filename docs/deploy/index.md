---
title: Choose where to run LyEve
description: "Compare the hosts LyEve runs on, pick a supported database, and find the step-by-step guide for each one."
sidebar:
  order: 1
---

LyEve ships as two container images. You run each one where it fits, next to a database you
provide.

| Image | What it is | Listens on |
|---|---|---|
| `ghcr.io/lyeve-labs/lyeve-core` | The engine: the Admin API and the Content API | `3001` (Admin API), `3002` (Content API) |
| `ghcr.io/lyeve-labs/lyeve-admin` | The admin console, a Node server that calls the engine for the browser | `3002` |

The engine is a long-running server. It keeps a pool of database connections open, holds its
signing key, and applies its migrations when it starts. A platform that starts a process per
request, such as Vercel Functions, Cloudflare Workers or Netlify Functions, cannot host it.
[Concepts](../concepts/index.md) explains what each API is for.

## Compare the hosts

| Host | Best for | What you give up | Guide |
|---|---|---|---|
| Docker | Trying LyEve, or one server you manage yourself | You wire the database, the proxy and TLS | [Docker images](docker.md) |
| Docker Compose | The whole stack on one server: engine, admin console, Postgres and TLS | One machine, so no failover | [Docker Compose](docker-compose.md) |
| Kubernetes | Several replicas, rolling deploys and a cluster you already run | More than one replica needs Redis, an S3 bucket and a shared signing key | [Kubernetes](kubernetes.md) |
| Google Cloud Run | An engine that scales to zero and bills per request | One API per service, cold starts, and a new signing key per instance | [Cloud Run](engine-cloud-run.md) |
| Fly.io | Both APIs public from one small Machine | Usage is billed, the Admin API answers on port 8443, and each Machine start signs everyone out unless a volume keeps the signing key | [Fly.io](engine-fly.md) |
| Railway | Both APIs from one service, with a volume for the signing key | No lasting free plan, usage is billed | [Railway](engine-railway.md) |
| Render | A free engine for a demo or staging | One API per service, and a free service spins down and signs everyone out | [Render](engine-render.md) |
| Free-tier stack | An engine and a database at $0 a month | Cold starts, tight connection limits, and no free host for the admin console | [Free-tier stack](free-tier-overview.md) |
| Stateless mode | A webhook relay in one container, with no database | No admin console, one tenant, and nothing kept across a restart | [Stateless mode](stateless-mode.md) |

:::tip[Not sure?]
- **Evaluating on a laptop:** [Docker Compose](docker-compose.md) runs everything, the admin
  console included.
- **Production on one server:** Docker Compose with your own Postgres backups.
- **Production with failover:** [Kubernetes](kubernetes.md), and read
  [Scaling](../scaling/index.md) first.
- **Spending nothing:** the [free-tier stack](free-tier-overview.md), for a demo or staging.
:::

## Supported databases

The engine stores everything in one database. It reads the type from the scheme of
`DATABASE_URL` and creates its tables on first start.

| Database | `DATABASE_URL` form |
|---|---|
| PostgreSQL | `postgres://user:pass@host:5432/lyeve?sslmode=require` (or `postgresql://`) |
| MySQL 8 or later | `mysql://user:pass@tcp(host:3306)/lyeve`, with the host in `tcp(...)` |
| SQL Server 2019 or later, and Azure SQL | `sqlserver://user:pass@host:1433?database=lyeve` |

SQL Server needs read committed snapshot isolation turned on before it takes traffic. See
[Installation](../getting-started/installation.md).

Use a dedicated database user. A production engine refuses to start as `postgres` on
PostgreSQL, `root` on MySQL or `sa` on SQL Server.

### SQLite, Turso and libSQL are not supported

The engine reads any URL whose scheme is not one of the above as PostgreSQL. A URL for SQLite,
Turso or libSQL therefore fails to connect, and the engine does not start. For a free database
with no server to run, use managed Postgres instead:

- [Supabase](database-supabase.md): free managed Postgres.
- [Neon](database-neon.md): free serverless Postgres that suspends when idle.

Both are a `DATABASE_URL` value and nothing more. For an engine with no database at all, see
[Stateless mode](stateless-mode.md).

## The admin console needs a server, not Vercel or Cloudflare Pages

The admin console is a long-running Node server. It ships as the container image
`ghcr.io/lyeve-labs/lyeve-admin`, and its source is public in
[`lyeve-admin`](https://github.com/lyeve-labs/lyeve-admin). Vercel and Cloudflare Pages run
neither a container image nor a long-running server, so neither runs the console as it ships.

Run the image as its own service on a host that runs containers, Railway and Render included.
The console's server sends every `/api` call to one base URL, `CORE_INTERNAL_URL`, so that URL
must route `/api/admin` to the engine's port `3001` and every other `/api` path to port `3002`.
A small proxy or an Ingress does that:

- [Docker Compose](docker-compose.md): the engine, the console, Postgres and a TLS proxy on one
  host.
- [Kubernetes](kubernetes.md): the same images on a cluster, routed by an Ingress.

[The admin image](docker.md#the-admin-image) lists the console's settings.

## Before go-live

- [Production checklist](production-checklist.md): every setting to confirm, one line each.
- [Configuration](../reference/configuration.md): every variable and its default.
- [Scaling](../scaling/index.md): more than one replica, Redis and load shedding.
