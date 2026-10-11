---
title: Database on Supabase
description: "Connect the engine to a free Supabase Postgres database through the session pooler or a direct connection."
sidebar:
  order: 9
---

Supabase offers a free managed PostgreSQL database. The engine needs only a standard Postgres
connection string and uses no Supabase feature beyond the database.

:::tip[Best for]
- **A free database for a demo or staging engine**, with backups and a dashboard handled for
  you.
:::

## Before you start

- A Supabase account with a project, and its database password.

## Get the connection string

Open the project's connection settings in the Supabase dashboard. It offers three strings:

| String | Host and port | Use it |
|---|---|---|
| Session pooler | `aws-0-<region>.pooler.supabase.com:5432`, user `postgres.<project-ref>` | Yes, from any engine host. |
| Direct connection | `db.<project-ref>.supabase.co:5432` | Only when your host can reach IPv6, and with a role you created. |
| Transaction pooler | port `6543` | No. It is built for short-lived serverless clients. |

```text
postgres://postgres.<project-ref>:<password>@aws-0-<region>.pooler.supabase.com:5432/postgres?sslmode=require
```

The engine keeps its own pool of long-lived connections, and while it applies migrations at
startup it holds a session lock that the transaction pooler does not keep. The session pooler
gives each engine connection a server connection of its own.

A production engine refuses to start when the database user is `postgres`. The session
pooler's user, `postgres.<project-ref>`, is accepted. For a direct connection, create a role for
the engine and put it in the URL.

## Secrets

Set the string as `DATABASE_URL` on the engine, through your host's secret store. Keep
`?sslmode=require`. Without it the driver may connect without TLS, and a production engine logs
a warning on every start.

## Connection limits

The session pooler serves a fixed number of server connections, which the dashboard shows as
the pool size. Keep `DATABASE_MAX_CONNECTIONS` under it, such as `5`, so a redeploy that runs
the old and new engine side by side still fits. The default is `25`.

## Health checks

The engine's `/readyz` and `/healthz` answer `503` while the database cannot be reached. Point
your host's checks at `/readyz`, as each engine host page shows.

## Verify

Start the engine and ask for readiness on either port:

```bash
curl https://<engine-url>/readyz
```

A `200` with `"status":"ok"` means the engine connected and applied its migrations.

## Next

- [Database on Neon](database-neon.md): another free managed Postgres.
- [Free-tier stack](free-tier-overview.md): the $0 recipe with an engine host.
- [Supported databases](index.md#supported-databases): why SQLite, Turso and libSQL are not an
  option.
