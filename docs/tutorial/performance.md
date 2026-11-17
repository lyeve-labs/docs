---
title: "24. Performance"
description: "Read the connection pool and the slowest endpoints, capture a flame graph, check the heap for a leak, size the goroutine engine, analyze slow queries, cache public reads and see idempotency keys."
sidebar:
  order: 24
covers:
  console:
    - "/admin/observability"
    - "/admin/observability/profiler"
    - "/admin/observability/profiler/capture"
    - "/admin/observability/profiler/memory"
    - "/admin/observability/goroutines"
    - "/admin/observability/queries"
    - "/admin/observability/cache"
    - "/admin/observability/cache/rules"
    - "/admin/observability/cache/stored"
    - "/admin/observability/idempotency"
  api:
    - "GET /api/admin/pool/health"
    - "GET /api/admin/debug/latency"
    - "GET /api/admin/debug/profiler/slowest"
    - "POST /api/admin/debug/profiler/flamegraph/{endpoint}"
    - "GET /api/admin/debug/profiler/memory"
    - "GET /api/admin/debug/goroutines/pool"
    - "PUT /api/admin/debug/goroutines/pool"
    - "GET /api/admin/query-log"
    - "GET /api/admin/query-log/analyze-pg"
    - "GET /api/admin/cache/stats"
    - "POST /api/admin/cache/rules"
    - "POST /api/admin/cache/flush"
    - "POST /api/v1/content/{schema}"
    - "GET /api/admin/idempotency/stats"
    - "DELETE /api/admin/idempotency/{key}"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson you look for what makes the engine slow: the database pool, the slowest
endpoints, where the CPU time goes, the heap, the queries, and what the cache and the
idempotency store hold. You need a super admin account, because the profiler and slow queries
open only for one, and a token in `TOKEN` for the API steps.
[Lesson 23](logs-and-errors.md) is the one before this.

Read a few trips first, so the screens have something to show.

## Read the overview

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Open **Insight > Observability**. **Connection pool** shows the database engine, the ping
latency, the goroutines, and the connections **Open**, **In use** and **Idle**, with a
**Healthy** or **Unhealthy** badge. **Wait count** that keeps growing means requests queue for
a connection. **Latency rankings** lists the endpoints with their count and the minimum,
average and maximum time, and **Rows** sets how many.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/pool/health \
  -H "Authorization: Bearer $TOKEN"

curl "http://localhost:3001/api/admin/debug/latency?top=10" \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![The Observability overview, with a healthy connection pool and the latency rankings.](./media/performance-overview.png)

## Find the slowest endpoints

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Open **Insight > Observability > Profiler**. The **Timings** tab lists the 20 slowest
   endpoints by p95.
2. Choose the path of `/api/v1/content/trip`. The page adds its methods and its most recent
   requests, each with its status, duration, memory allocated and goroutines.
3. **By plugin** below shows where the time went by feature.

**Reset** deletes the samples and the memory snapshots and starts again.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/debug/profiler/slowest \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

## Capture a flame graph

> **Included free** on every install.

A flame graph shows where the CPU time goes while you send traffic.

<div class="paths">
<div data-path="console">

1. On **Profiler**, choose the **Profiles** tab.
2. Under **Flamegraph**, enter `/api/v1/content/trip` in **Endpoint** and `5` in **Window**.
3. Choose **Capture**, and read a few trips in another tab while it runs. The graph appears
   when the window ends.

The endpoint is a label only, because the profile covers the whole process. **pprof export**
downloads a profile for the standard Go tools, and **CPU window** sets how long a CPU profile
samples.

</div>
<div data-path="api">

```bash
curl -X POST "http://localhost:3001/api/admin/debug/profiler/flamegraph/api/v1/content/trip?duration_sec=5" \
  -H "Authorization: Bearer $TOKEN"
```

Save the `svg` field to a file and open it in a browser.

</div>
</div>

![Entering the trip endpoint and a five second window on the Profiles tab, choosing Capture and the flame graph appearing.](./media/performance-flamegraph.mp4)

## Check the heap for a leak

> **Included free** on every install.

<div class="paths">
<div data-path="console">

On **Profiler**, choose the **Memory** tab. The badge reads **Steady** or **Leak likely**, beside
**Heap in use**, **Heap objects**, **Allocated total**, **GC cycles**, **Goroutines** and
**Heap slope**. Snapshots are taken on a schedule, so the trend fills in over time.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/debug/profiler/memory \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![The Memory tab with a steady heap and its snapshot figures.](./media/performance-memory.png)

## Size the goroutine engine

> **Included free** on every install, to read. Changing a setting on a running engine needs a license with the `goroutine-engine-pro` feature. See [pricing](https://lyeve.com/pricing/).

<div class="paths">
<div data-path="console">

1. On **Insight > Observability**, choose the **Goroutine engine** tab.
2. Read the worker pool: **Pool size**, **Active**, **Queued**, **Completed**, **Failed** and
   **Dropped**. A steady queue means the pool is too small.
3. With the license, enter a new size in **Pool size** under **Resize the pool** and choose
   **Apply**.

The tab also shows **Goroutines by owner**, and sets the parallel engine and the async hooks.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/debug/goroutines/pool \
  -H "Authorization: Bearer $TOKEN"
```

With the license, resize the pool:

```bash
curl -X PUT http://localhost:3001/api/admin/debug/goroutines/pool \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"size": 200}'
```

</div>
</div>

On a free install the readings are the same, the controls are turned off, and a note says tuning
the goroutine engine is not enabled on this engine. A change made here lasts until the next restart.
[Concurrency tuning](../features/concurrency-tuning.md#settings) names the settings that keep it.

![The Goroutine engine tab on a free install, with the numbers and the resize controls not enabled.](./media/performance-goroutines-free.png)

![The Goroutine engine tab on a licensed install, with the pool size ready to change.](./media/performance-goroutines-licensed.png)

## Analyze slow queries

> **Requires** a license with the `query-monitor` feature. See [pricing](https://lyeve.com/pricing/).

<div class="paths">
<div data-path="console">

1. Open **Insight > Observability > Slow queries**. **Captured statements** lists each slow
   entry with the time it **Took**, its **Rows**, its **Caller** and when it was **Captured**.
2. Choose **Analyze postgres**, or the button for your database. **Index suggestions** lists
   each **Table**, its **Columns**, **Why**, and the **DDL** that would create the index.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/query-log?limit=5" \
  -H "Authorization: Bearer $TOKEN"

curl "http://localhost:3001/api/admin/query-log/analyze-pg?limit=5" \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

Without the license the page says **The query monitor is not included in this build**.
[Prepare the database](../features/slow-query-analysis.md#prepare-the-database) says what each
database needs before plans can be read.

![The Slow queries address on a free install, which says the query monitor is not included.](./media/performance-queries-free.png)

![Slow queries on a licensed install, with the captured statements and index suggestions.](./media/performance-queries-licensed.png)

## Cache public reads

> **Included free** on every install, with up to 10 response cache rules per tenant. More rules need a license with the `cache-pro` feature. See [pricing](https://lyeve.com/pricing/).

A rule keeps the answers of a public read route, one that answers without a credential. The
[public delivery](../features/content-delivery.md) routes are such routes, so once `trip` is
marked for public delivery, a rule can serve its trips from the cache.

<div class="paths">
<div data-path="console">

1. Open **Insight > Observability > Cache**. The **Overview** tab shows the **Hit rate**,
   **Entries**, **Evicted** and **Misses**, and each provider with its circuit state.
2. Choose the **Response cache rules** tab, then **New rule**.
3. Enter `/api/v1/delivery/**` in **Route**, `300` in **Time to live, in seconds** and `trip`
   in **Tags**, so a trip change clears it. Keep **Serve from the cache** on.
4. Choose **Create**. The rule reads **Serving**.

The **Stored** tab lists what the cache holds. **Flush the cache** there empties it.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/cache/rules \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pattern": "/api/v1/delivery/**", "ttl_seconds": 300, "tags": ["trip"]}'

curl http://localhost:3001/api/admin/cache/stats \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/cache/flush \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![Opening New rule on Response cache rules, entering the delivery route, a time to live and the trip tag, and creating it.](./media/performance-cache-rule.mp4)

A request that carries a token, an API key or a cookie is never served from the cache.
[What is never cached](../features/caching.md#what-is-never-cached) lists the rest.

## See idempotency keys

> **Included free** on every install.

A client that sends an `Idempotency-Key` can retry a write without making it twice. Create a trip
with a key, with the fields your `trip` type requires:

```bash
curl -X POST http://localhost:3002/api/v1/content/trip \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: trip-lisbon-001" \
  -d '{"data": {"title": "Lisbon in four days"}}'
```

Send it again, and the answer is the same entry with `Idempotency-Replayed: true`.

<div class="paths">
<div data-path="console">

Open **Insight > Observability > Idempotency**. **Keys held**, **With a stored response** and
**In flight** count the keys. **Keys** lists `trip-lisbon-001` with its request, **Stored** and
one replay. A key whose request never finished can be released from its row, so the next
request with it runs again.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/idempotency/stats \
  -H "Authorization: Bearer $TOKEN"
```

To let the next request with a stored key run again, delete it:

```bash
curl -X DELETE http://localhost:3001/api/admin/idempotency/trip-lisbon-001 \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

## Next

[25. Captures, synthetic checks and telemetry](captures-and-telemetry.md) records and replays
requests and watches the site from outside. For depth, read
[Request profiling](../features/request-profiling.md),
[Slow query analysis](../features/slow-query-analysis.md),
[Caching](../features/caching.md), [Idempotent requests](../features/idempotent-requests.md),
[Concurrency tuning](../features/concurrency-tuning.md) and [Scale and tune](../scaling/index.md).
