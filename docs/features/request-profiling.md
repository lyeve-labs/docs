---
title: Request profiling
description: Find the slowest endpoints, watch heap growth, download CPU, heap and goroutine profiles, and tune garbage collection on a running instance.
sidebar:
  order: 53
---

> **Included free** on every install.

Request profiling shows where your instance spends its time and memory, with
no agent to install and no redeploy. It times every request to the Admin API
and the Content API, ranks endpoints by latency, watches the heap for signs of
a leak, and hands you profiles to open on your own computer.

:::tip[Best for]
- **Finding which endpoint is slow.** Rank every endpoint by p95, then look at
  one endpoint's recent requests.
- **Seeing where the CPU time goes.** Record a CPU profile or a flame graph for
  a few seconds while the slow traffic runs.
- **Telling a memory leak from normal growth.** The heap trend says whether
  the heap keeps growing, and how fast.
- **Finding a goroutine leak.** Goroutine counts are grouped by the feature
  that started them.
- **Trading memory for CPU.** Change how often garbage collection runs on the
  live process, with no restart.
:::

:::note[Use something else when]
- You want latency and error rates over days on a dashboard.
  [Metrics export](metrics-export.md) sends them to your monitoring stack.
- The time goes into the database. [Slow query analysis](slow-query-analysis.md)
  records the slow statements and their plans.
- You want to know why one request failed rather than why many are slow.
  [Logs](logs.md) find it by request id.
:::

## How it works

| Tool | Who | What it gives you |
|---|---|---|
| **Profiler** | Super admin | The last 10,000 requests, ranked by endpoint, method and feature, a heap trend, profile downloads and flame graphs. |
| **Latency ranking** | Admin | The slowest endpoints by p50, p95 and p99, for an admin who is not a super admin. |
| **Standard profiles** | Super admin | The usual pprof URLs, for tools that expect them. |
| **Goroutine counts and garbage collection** | Admin reads, super admin writes | Live counts, and the collector's settings. |

Everything here is held in memory. It covers the whole process, not one
tenant, and a restart clears it. With more than one replica, each replica
answers for itself.

A **profile** is a file that lists where the process spent CPU time, or what
it holds in memory, by function. You open it with `go tool pprof`, which comes
with the Go toolchain. You need Go on your own computer only, not on the
server, and you need no Go knowledge to read the result:
`go tool pprof -http=:8080 cpu.pb.gz` opens it in a browser, flame graph
included.

## Try it

You need a super admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md) shows how to get one.

1. Rank the slowest endpoints:

   ```bash
   curl http://localhost:3001/api/admin/debug/profiler/slowest \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   {
     "slowest": [
       {
         "endpoint": "/api/v1/content/post",
         "method": "GET",
         "count": 1520,
         "avg_duration_ns": 8100000,
         "p50_duration_ns": 6200000,
         "p95_duration_ns": 21400000,
         "p99_duration_ns": 48000000,
         "max_duration_ns": 93000000,
         "avg_alloc_bytes": 0,
         "total_alloc_bytes": 0,
         "last_seen": "2026-10-01T09:30:12Z"
       }
     ]
   }
   ```

   Durations are in nanoseconds, so `21400000` is 21.4 ms.

2. Look at one endpoint's recent requests. The path follows `endpoint`:

   ```bash
   curl "http://localhost:3001/api/admin/debug/profiler/endpoint/api/v1/content/post?limit=5" \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer has `methods`, each with its `stats` and `recent_entries`.

3. Record a CPU profile for five seconds while the slow traffic runs, then open
   it:

   ```bash
   curl -X POST "http://localhost:3001/api/admin/debug/profiler/profile/cpu?duration_sec=5" \
     -H "Authorization: Bearer $TOKEN" -o cpu.pb.gz
   go tool pprof -top cpu.pb.gz
   ```

   `-top` lists the functions that used the most CPU. Use `-http=:8080` in its
   place for the browser view.

4. Check the heap trend:

   ```bash
   curl http://localhost:3001/api/admin/debug/profiler/memory \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer has `snapshots`, `slope_bytes_per_sec` and `leak_likely`.

5. In the admin console, open **Insight > Observability > Profiler** to see the
   same rankings and heap trend on screen.

## Rank the slowest endpoints

- `GET /api/admin/debug/profiler/endpoints` lists every endpoint and method
  with count, average, p50, p95, p99 and maximum duration, average and total
  allocation, and when it was last seen.
- `GET /api/admin/debug/profiler/slowest` gives the 20 with the highest p95.
- `GET /api/admin/debug/profiler/plugins` groups the same figures by the
  feature that served each request. Requests the instance served without a
  feature are grouped as `core`.
- `GET /api/admin/debug/profiler/endpoint/{path}` gives one endpoint with its
  recent requests. `{path}` is the request path as sent, slashes included.
  `limit` (default 100, at most 1000) and `offset` page the requests.

An admin who is not a super admin can read the latency ranking instead:

```bash
curl "http://localhost:3001/api/admin/debug/latency?top=10" \
  -H "Authorization: Bearer $TOKEN"
```

It ranks by p95 and reports `min_us`, `avg_us`, `p50_us`, `p95_us`, `p99_us`
and `max_us` in microseconds. `top` defaults to 20, at most 1000. It tracks up
to 200 distinct paths with 500 samples each, and `tracker` in the answer says
how full it is.

## See where the CPU time goes

`POST /api/admin/debug/profiler/profile/cpu?duration_sec=10` records for that
many seconds, from 1 to 25, default 5. The request stays open for the whole
time. One CPU profile or flame graph runs at a time, and a second request waits
for the first.

A flame graph comes back ready to view:

```bash
curl -X POST "http://localhost:3001/api/admin/debug/profiler/flamegraph/api/v1/content/post?duration_sec=5" \
  -H "Authorization: Bearer $TOKEN"
```

```json
{
  "endpoint": "/api/v1/content/post",
  "duration_sec": 5,
  "svg": "<svg xmlns=\"http://www.w3.org/2000/svg\">",
  "profile_type": "cpu",
  "captured_at": "2026-10-01T09:31:00Z"
}
```

Save `svg` to a file and open it in a browser. The endpoint in the path is a
label only: the graph covers everything the process did during the window.

## Check for a memory leak

The heap is sampled every 30 seconds, and a day of samples is kept.
`leak_likely` is `true` when, over the last 60 samples (about half an hour),
the heap grows faster than 1 KiB per second and more than 70% of the samples
grew. `leak_reason` explains a likely leak, or
says there is not enough data yet: the verdict needs five samples, about two
and a half minutes after a start.

To see what holds the memory, download a heap profile:

```bash
curl -X POST http://localhost:3001/api/admin/debug/profiler/profile/heap \
  -H "Authorization: Bearer $TOKEN" -o heap.pb.gz
go tool pprof -top heap.pb.gz
```

`profile/allocs` gives every allocation since start, and `profile/goroutine`
every running goroutine with its stack.

:::danger
A heap or allocs profile holds raw process memory, which can include content,
credentials and personal data. Treat a downloaded profile like a database
dump. Every heap or allocs download from the profiler is written to the server
log with the caller's user id and address. Set `PROFILER_NO_HEAP_EXPORT=true`
in production to refuse them.
:::

## Count goroutines

A goroutine is a task running inside the process. A count that only goes up is
a leak.

```bash
curl http://localhost:3001/api/admin/debug/goroutines \
  -H "Authorization: Bearer $TOKEN"
```

```json
{
  "total": 1,
  "runtime_total": 43,
  "by_source": { "review-sla-check": 1 },
  "by_owner": { "review": { "total": 1, "oldest": "22m33.18s" } },
  "max_goroutines": 50000,
  "leak_threshold": "2m0s"
}
```

`runtime_total` counts every goroutine in the process. `total`, `by_source` and
`by_owner` count the background tasks the instance tracks, by name and by the
feature that started them, so a leak names its feature. An admin can read it.
The worker pool and the other concurrency views live under
[concurrency tuning](concurrency-tuning.md).

## Profile with the standard pprof tools

The standard Go profile URLs are served under `/api/admin/debug/pprof/` for a
super admin. The index there lists every profile:

```bash
curl http://localhost:3001/api/admin/debug/pprof/heap \
  -H "Authorization: Bearer $TOKEN" -o heap.pb.gz
curl "http://localhost:3001/api/admin/debug/pprof/profile?seconds=10" \
  -H "Authorization: Bearer $TOKEN" -o cpu.pb.gz
```

They give the same files as the profiler's exports, plus `block`, `mutex`,
`threadcreate`, `trace`, `cmdline` and `symbol`. They take no setting:
`PROFILER_NO_HEAP_EXPORT` does not apply to them, and the super admin role is
the only control.

## Tune garbage collection

`GET /api/admin/debug/gc-config` shows the `GOGC` and `GOMEMLIMIT` environment
values the process started with, and `memory_limit`, the limit in force in
bytes. With no limit set, `memory_limit` is `9223372036854775807`.

`GOGC` says how far the heap may grow before the next collection, as a
percentage of what survived the last one. The default, `100`, collects when
the heap doubles. A higher value uses more memory and less CPU, a lower one the
reverse. A super admin can change it on the running process:

```bash
curl -X POST http://localhost:3001/api/admin/debug/gc-config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"gogc": 200}'
```

```json
{ "previous_gogc": 100, "current_gogc": 200 }
```

:::caution
The change applies to this replica until it restarts. Set the `GOGC`
environment variable to keep it. A value of `-1` turns garbage collection off,
which lets memory grow until the process is killed. For the memory ceiling the
instance sizes itself against, see [scale and tune](../scaling/index.md).
:::

## Settings

Each profiler setting can be an environment variable or saved in the admin
console. A saved setting takes effect on the next request.

| Variable | What it does | Default |
|---|---|---|
| `PROFILER_RING_SIZE` | How many recent requests are kept, from 1 to 100000. A value outside that range is ignored. Changing it clears the requests already recorded. | `10000` |
| `PROFILER_CAPTURE_MEMSTATS` | Record allocations, heap and goroutine count for each request. This pauses the process for about 50 to 200 microseconds per request, so leave it off unless you are chasing memory. The allocation fields stay `0` while it is off. | `false` |
| `PROFILER_NO_HEAP_EXPORT` | Refuse the profiler's heap and allocs exports with `403`. Recommended in production. | `false` |

The profiler runs on every install. If you set `LYEVE_PLUGINS` to choose which
features start, include `profiler` in it. The latency ranking, the goroutine
counts, garbage collection and the standard pprof URLs are always served. See
[licensing and tiers](../concepts/licensing-and-tiers.md).

## Routes

<details>
<summary>Profiler routes (super admin)</summary>

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/admin/debug/profiler/endpoints` | Figures per endpoint and method. |
| `GET` | `/api/admin/debug/profiler/endpoint/{path}` | One endpoint with its recent requests, with `limit` and `offset`. |
| `GET` | `/api/admin/debug/profiler/plugins` | Figures per feature. |
| `GET` | `/api/admin/debug/profiler/slowest` | The 20 endpoints with the highest p95. |
| `GET` | `/api/admin/debug/profiler/memory` | Heap samples, growth rate and the leak verdict. |
| `POST` | `/api/admin/debug/profiler/profile/cpu` | Download a CPU profile. `duration_sec` from 1 to 25, default 5. |
| `POST` | `/api/admin/debug/profiler/profile/goroutine` | Download a goroutine profile. |
| `POST` | `/api/admin/debug/profiler/profile/heap` | Download a heap profile. |
| `POST` | `/api/admin/debug/profiler/profile/allocs` | Download an allocs profile. |
| `POST` | `/api/admin/debug/profiler/flamegraph/{path}` | Record CPU for `duration_sec` and return an SVG flame graph. |
| `POST` | `/api/admin/debug/profiler/reset` | Clear the recorded requests and heap samples. |

</details>

<details>
<summary>Other debug routes</summary>

| Method | Path | Who | Purpose |
|---|---|---|---|
| `GET` | `/api/admin/debug/latency` | Admin | The slowest endpoints, with `top`. |
| `GET` | `/api/admin/debug/goroutines` | Admin | Goroutine counts. |
| `GET` | `/api/admin/debug/gc-config` | Admin | Garbage collection settings. |
| `POST` | `/api/admin/debug/gc-config` | Super admin | Set `{"gogc": <n>}` on the running process. |
| `GET` | `/api/admin/debug/pprof/` | Super admin | The index of standard profiles. |
| `GET` | `/api/admin/debug/pprof/{heap,goroutine,allocs,block,mutex,threadcreate,profile,trace,cmdline,symbol}` | Super admin | One standard profile. |
| `GET`, `PUT` | `/api/admin/debug/goroutines/{status,pool,parallel,async-hooks}` | Admin reads, super admin writes | See [concurrency tuning](concurrency-tuning.md). |

</details>

## Errors

| Status | Message | Cause |
|---|---|---|
| `403` | `heap profile export is disabled` | `PROFILER_NO_HEAP_EXPORT` is on. |
| `404` | `no data for endpoint: <path>` | Nothing was recorded for that path since the last start or reset. |
| `500` | `flamegraph generation failed` | The profile could not be rendered. The server log has the cause. |

## Related

- [Slow query analysis](slow-query-analysis.md): when the time is spent in the
  database.
- [Concurrency tuning](concurrency-tuning.md): worker pools and their live
  views.
- [Metrics export](metrics-export.md): latency over time in your dashboards.
- [Scale and tune](../scaling/index.md): memory limits, the connection pool and
  more replicas.
