---
title: Slow query analysis
description: Find the requests that are slow because of the database, read their query plans and get index suggestions on PostgreSQL, MySQL and SQL Server.
sidebar:
  order: 55
---

> **Requires** a license with the `query-monitor` feature. See [pricing](https://lyeve.com/pricing/).

Slow query analysis watches how long each request takes. When one passes the
threshold, it records the request, reads the statements that cost the
database the most on average, and stores each one the database can plan beside
it, with its plan and the indexes that would help. Use it to find out why a content type or an
endpoint got slow, and what to index.

:::tip[Best for]
- **Finding out why a listing got slow.** A slow request is stored with its
  method, path and time, and the database's most expensive statements on
  average are stored beside it with their plans.
- **Deciding what to index.** On PostgreSQL and MySQL, a plan that scans a
  whole table comes with a ready `CREATE INDEX` statement.
- **Finding the heaviest statements on SQL Server.** The SQL Server view ranks
  cached queries by CPU time, reads, duration and executions, and lists the
  indexes the server itself says are missing.
:::

:::note[Use something else when]
- The time is spent in the instance, not in the database.
  [Request profiling](request-profiling.md) ranks endpoints and records CPU
  profiles.
- You want query latency over time in your own dashboards.
  [Metrics export](metrics-export.md) sends the database pool metrics there.
:::

## How it works

- Timing a request adds no database work. The analysis runs after the answer
  is sent, so it never slows the request it records.
- Requests are timed and analyzed after the fact. Individual queries are not
  intercepted. The request is stored as its method and path. Beside it go the
  three statements with the highest average time since the database's
  statistics were last reset: `mean_exec_time` in `pg_stat_statements` on
  PostgreSQL, `AVG_TIMER_WAIT` in `performance_schema` on MySQL. They are the
  most expensive statements on average, not the ones that request ran. Only
  SQL Server takes the statement it ran most recently.
- A plan is read with `EXPLAIN`, without the statement's values. On
  PostgreSQL, `pg_stat_statements` replaces every value with a placeholder
  such as `$1`, and PostgreSQL refuses to plan a statement that still holds
  one, so such a statement is skipped. On engine v0.56.0 that covers the
  statements LyEve runs for content, so on PostgreSQL the plans you see come
  from statements written without values.
- A server-sent event stream or a WebSocket is never captured, because it
  stays open as long as its client does, so its time measures the client.
  [Request profiling](request-profiling.md) leaves streams out of its timings
  too.
- Literals are removed from stored SQL and plans. Strings and numbers become
  `?`, so no customer data or secret is kept.
- Each entry belongs to the tenant whose request triggered it. Deleting a
  tenant deletes its entries.

A content type's table is its name with a leading underscore, so the
`post` type is stored in `_post`. That is the name you see in plans and
suggestions.

## Prepare the database

The database has to report its expensive statements:

| Database | What to do |
|---|---|
| PostgreSQL | Install `pg_stat_statements`: add it to `shared_preload_libraries`, restart PostgreSQL, run `CREATE EXTENSION pg_stat_statements;`, then restart LyEve. |
| MySQL | Turn on `performance_schema`. |
| SQL Server | Nothing. |

LyEve checks for the extension once, so until it restarts the analysis keeps
answering `source_missing`. Without the PostgreSQL extension or MySQL's
`performance_schema`, a slow request is still recorded with its method, path
and duration, but with no statement and no plan.

## In the admin console

Open **Insight > Observability > Slow queries**. Only a super admin can open
it. **Sampled** counts the recorded entries, beside the slowest and the median
time on the page. **Captured statements** lists each entry with the time it
**Took**, its **Rows**, its **Caller** and when it was **Captured**, and the
search box above it narrows the page. **Analyze postgres**, **Analyze mysql**
and **Analyze mssql** run the plan analysis when you ask, and list what they
find under **Index suggestions**.

![The Slow queries page with the sampled count, the slowest and median times, the three Analyze buttons and the captured statements, each with the time it took.](images/slow-query-analysis.png)

In the picture every entry is a request, read as its method and path. A
statement stored beside a request reads as its SQL, with a row count and no
caller.

## Try it

For the console steps, you are signed in to the admin console as a super
admin. For the API steps, you need a super admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md) shows how to get one.

**1. List the recorded slow requests.**

<div class="paths">
<div data-path="console">

Open **Insight > Observability > Slow queries**. **Captured statements** lists
the newest entries first. Type part of a path or a statement in the box that
reads **Statement, caller or tenant on this page** to find one.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/query-log?limit=5" \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `{"data": [...], "total_count": n, "limit": 5, "offset": 0}`.
A quiet instance answers an empty `data`.

</div>
</div>

**2. Read the plan analysis** on PostgreSQL or MySQL.

<div class="paths">
<div data-path="console">

Press **Analyze postgres**, or **Analyze mysql** on MySQL. Each suggestion
lists its **Table**, **Columns**, **Why** and the **DDL** that would create
the index. When no entry carries a plan, the panel says why, for example
that `pg_stat_statements` is there and none of the recent slow requests
captured a plan. **Clear** hides the result.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/query-log/analyze-pg?limit=5" \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `{"engine": "postgres", "entries": [...], "plans_read": n}`,
where `plans_read` counts the entries that carry a plan. On MySQL, call
`analyze-mysql` instead. When `plans_read` is `0`, `no_plans_reason` says
why:

| `no_plans_reason` | Meaning |
|---|---|
| `other_dialect` | The instance runs another database. |
| `capture_disabled` | `QUERY_MONITOR_DISABLE_PG_STAT_CAPTURE` is set. |
| `source_missing` | `pg_stat_statements` is not installed, or MySQL's `performance_schema` digest table is missing. |
| `source_unreadable` | Checking for the source failed. |
| `none_captured` | The source is there, and no captured statement carries a plan yet. |

</div>
</div>

**3. Read SQL Server's own ranking.**

<div class="paths">
<div data-path="console">

Press **Analyze mssql**. The panel lists the indexes from SQL Server's
missing-index views. The rankings by CPU time, reads, duration and executions
are in the API answer only. On an instance that runs another database, the
panel says that SQL Server suggested no index.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/query-log/analyze-mssql?top_n=10" \
  -H "Authorization: Bearer $TOKEN"
```

The answer has `top_cpu_time`, `top_io_reads`, `top_duration`,
`top_executions` and `missing_indexes`. On another database every list is
empty.

</div>
</div>

## Read a plan and its suggestions

This entry from `analyze-pg` is a statement stored beside a slow request. It
is the most expensive statement on average at that moment, not the request's
own: the request was a content list, and this statement was run separately.
The plan is PostgreSQL's JSON plan, shortened here:

```json
{
  "id": "aa2cf2f8-5185-4b61-8a0d-aacee020a44c",
  "tenant_id": "default",
  "query_hash": "3d6957c1be2d2e8f",
  "query_text": "SELECT id, title FROM _post WHERE updated_at >= current_date ORDER BY updated_at DESC",
  "plan_text": "[\n  {\n    \"Plan\": {\n      \"Node Type\": \"Gather Merge\", ...",
  "duration_ms": 1367.786,
  "rows_returned": 3159,
  "engine": "postgres",
  "captured_at": "2026-10-23T17:34:23.982351Z",
  "analysis": {
    "has_seq_scan": true,
    "rows_estimated": 836,
    "index_suggestions": [
      {
        "table": "_post",
        "columns": ["updated_at"],
        "reason": "Seq scan on _post with filter \"(updated_at >= CURRENT_DATE)\" (est. rows: 418)",
        "ddl": "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx__post_updated_at ON _post (updated_at)"
      },
      {
        "table": "",
        "columns": ["updated_at DESC"],
        "reason": "Sort on  ([updated_at DESC]): index would eliminate sort step",
        "ddl": "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx__updated_at DESC ON  (updated_at DESC)"
      }
    ]
  }
}
```

`has_seq_scan` means the database read the whole table. A suggestion names the
table, the columns and why, and `ddl` is the statement that would create the
index. Here the first suggestion is the one to act on: the statement filters
`_post` on `updated_at`, and an index on that column lets the database read
only the recent rows. On engine v0.56.0 a suggestion for a sort names no
table, and its `ddl` does not run, so skip a suggestion whose `table` is
empty.

`duration_ms` is the time of the request the statement was stored beside, not
the statement's own time. The request entry carries the plan of the first
statement beside it that earned a suggestion.

:::caution
Review a suggested index before you create it. A suggestion reads one plan and
does not know your write load, and every index slows writes to its table.
:::

## Settings

| Variable | What it does | Default |
|---|---|---|
| `QUERY_MONITOR_SLOW_THRESHOLD` | A request at or above this time is analyzed, such as `250ms` or `1s`. | `100ms` |
| `QUERY_MONITOR_DISABLE_PG_STAT_CAPTURE` | Set `true` to stop reading `pg_stat_statements`. | `false` |

On PostgreSQL, `pg_stat_statements` covers every statement the database server
ran, for every tenant. On an install with several tenants, the statement stored
beside one tenant's slow request can be another tenant's. Set
`QUERY_MONITOR_DISABLE_PG_STAT_CAPTURE=true` if that matters to you. MySQL and
SQL Server are not affected.

If you set `LYEVE_PLUGINS` to choose which features start, include
`query-monitor` in it. See
[licensing and tiers](../concepts/licensing-and-tiers.md).

## Troubleshooting

- **No statement is ever stored beside a request.** The database is not
  reporting its queries. Install `pg_stat_statements` on PostgreSQL or turn on
  `performance_schema` on MySQL, then restart LyEve. On PostgreSQL, a
  statement that holds a placeholder is never planned, as
  [How it works](#how-it-works) says.
- **Nothing is recorded.** No request has reached the threshold. Lower
  `QUERY_MONITOR_SLOW_THRESHOLD` while you investigate, and restart.

## Routes

All routes are read-only and need a super admin.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/admin/query-log` | Recorded slow requests, newest first. `limit` (default 50, at most 500) and `offset`. |
| `GET` | `/api/admin/query-log/analyze-pg` | Recent PostgreSQL entries with plan analysis and index suggestions. `limit` (default 20). |
| `GET` | `/api/admin/query-log/analyze-mysql` | The same for MySQL. `limit` (default 20). |
| `GET` | `/api/admin/query-log/analyze-mssql` | SQL Server's most expensive cached queries. `top_n` (default 20). |

## Errors

| Status | Message | Cause |
|---|---|---|
| `400` | `reqparse: invalid pagination parameter "limit": -1` | `limit`, `offset` or `top_n` is negative or not a number. |
| `402` | `payment_required` | The license no longer carries `query-monitor`. |
| `503` | `failed to load query log`, `failed to analyze queries`, `failed to analyze MySQL slow queries` or `failed to analyze MSSQL query store` | The database could not be read. |
| `503` | `no database connection available` | The instance has no database connection to read from. |

## Related

- [Request profiling](request-profiling.md): when the time is spent in the
  instance.
- [Scale and tune](../scaling/index.md): the connection pool and read
  replicas.
- [Metrics export](metrics-export.md): pool and latency metrics in your
  dashboards.
