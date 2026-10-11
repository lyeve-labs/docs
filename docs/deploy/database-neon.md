---
title: Database on Neon
description: "Connect the engine to a free Neon Postgres database, pick the right connection string, and plan for compute that suspends when idle."
sidebar:
  order: 10
---

Neon offers a free serverless PostgreSQL database. The engine treats it as a plain Postgres
server, so the only thing to set up is the connection string.

:::tip[Best for]
- **A free database for a demo or staging engine**, with no server to run.
- **Branching a copy of the data** for a test environment, which Neon does on its side.
:::

:::caution[What you give up]
Neon's free plan suspends the database after a period with no queries. The data stays, but the
first query after that waits for it to resume. On an engine host that also spins down, the
first request after idle time pays for both.
:::

## Before you start

- A Neon account with a project and a database.
- A role for the engine other than `postgres`. A production engine refuses to start as
  `postgres`.

## Get the connection string

Open your project's connection details in the Neon console. Neon offers a pooled string, whose
host contains `-pooler`, and a direct one. Use the direct string:

```text
postgres://<role>:<password>@<endpoint>.<region>.aws.neon.tech/<dbname>?sslmode=require
```

The pooled endpoint runs in transaction mode, which hands each transaction to whichever server
connection is free. The engine keeps its own pool of long-lived connections, and while it
applies migrations at startup it holds a session lock that a transaction-mode pooler does not
keep.

## Secrets

Set the string as `DATABASE_URL` on the engine, through your host's secret store. Neon requires
TLS, so keep `?sslmode=require`.

## Connection limits

Keep `DATABASE_MAX_CONNECTIONS` small, such as `5`, so the engine stays under the plan's
connection limit even while a redeploy runs the old and new engine side by side. The default is
`25`.

## Health checks

The engine's `/readyz` answers `503` while the database cannot be reached, and `/healthz` does
too. A host that restarts the container on a failed liveness check may restart it while Neon
resumes. Give the check a grace period of at least the time Neon takes to resume.

## Verify

Start the engine and ask for readiness on either port:

```bash
curl https://<engine-url>/readyz
```

A `200` with `"status":"ok"` means the engine connected and applied its migrations. If the
first request after idle time is slow, check whether the container, Neon's compute, or both were
resuming.

## Next

- [Database on Supabase](database-supabase.md): another free managed Postgres.
- [Free-tier stack](free-tier-overview.md): the $0 recipe with an engine host.
- [Supported databases](index.md#supported-databases): why SQLite, Turso and libSQL are not an
  option.
