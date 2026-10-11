---
title: Metrics export
description: Send your instance's metrics to Prometheus Pushgateway, an OTLP collector, Datadog or New Relic, and let each tenant scrape its own request metrics.
sidebar:
  order: 72
---

> **Included free** on every install, scheduled export included. A metrics destination of a tenant's own needs a license with the `multitenant-customization` feature. See [pricing](https://lyeve.com/pricing).

Metrics export sends your instance's request, database pool and runtime
metrics to the monitoring backend you already run, every 30 seconds by
default. It also serves a scrape route where a super admin reads every metric
and a tenant admin reads only their own tenant's request metrics.

:::tip[Best for]
- **Dashboards in the stack you already run.** Push to Prometheus
  Pushgateway, an OTLP collector, Datadog or New Relic without opening a
  scrape port.
- **Alerting on error rate and latency.** `lyeve_requests_total` counts
  requests by status class, and `lyeve_request_duration_seconds` is a latency
  histogram.
- **Catching a database pool before it runs dry.** The `lyeve_db_pool_*`
  metrics show connections in use, waits and utilization.
- **Giving each tenant its own numbers.** A tenant admin scrapes only the
  samples labeled with their tenant, or with `multitenant-customization` has
  them pushed to the tenant's own collector.
:::

:::note[Use something else when]
- You want to know which endpoint is slow right now, and where the time goes.
  [Request profiling](request-profiling.md) ranks routes by latency.
- You want traffic per endpoint, tenant and client in the console.
  [API analytics](api-analytics.md) charts it.
- You want to know why one request failed. [Logs](logs.md) find it by
  request id.
:::

## How it works

There are three ways to get the same metrics out.

| Way | Who reads it | Use it when |
|---|---|---|
| Prometheus scrapes `GET /api/admin/metrics` | Prometheus, with `METRICS_TOKEN` or a super admin token | You run Prometheus and it can reach the instance. `METRICS_TOKEN` is in the [configuration reference](../reference/configuration.md#observability). |
| Scrape `GET /api/admin/telemetry/metrics` | A super admin sees everything, a tenant admin sees their tenant | Each tenant needs its own view. |
| An exporter pushes on a schedule | Your backend | The backend cannot scrape the instance, or it is a hosted service. |
| A tenant destination pushes on the same schedule | The tenant's own collector | A tenant wants its numbers in its own monitoring. Needs `multitenant-customization`. |

Every export carries the same metrics:

- Process and Go runtime metrics.
- Database pool metrics, `lyeve_db_pool_*`.
- Request metrics: `lyeve_requests_total`, `lyeve_request_duration_seconds`
  and `lyeve_requests_in_flight`. Each carries a `tenant` label with the
  tenant slug, or `-` when the request named no tenant.
- Content cache counters, `lyeve_content_list_cache_hits_total` and
  `lyeve_content_list_cache_misses_total`.
- `lyeve_build_info`, labeled with version and database dialect,
  `lyeve_plugin_count` and `lyeve_runtime_seconds`.

Traces are not part of this feature. To send traces, see
[tracing under load](../scaling/index.md#tracing-under-load).

## Try it

You need an admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md) shows how to get one.

1. Scrape the request metrics:

   ```bash
   curl -s http://localhost:3001/api/admin/telemetry/metrics \
     -H "Authorization: Bearer $TOKEN" | grep '^lyeve_requests_total'
   ```

   ```text
   lyeve_requests_total{code="2xx",method="GET",plugin="core",tenant="default"} 27
   lyeve_requests_total{code="4xx",method="GET",plugin="core",tenant="default"} 3
   ```

2. List the exporters. On a fresh install none is configured:

   ```bash
   curl http://localhost:3001/api/admin/telemetry/exporters \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   { "data": [], "limit": 50, "offset": 0, "total_count": 0 }
   ```

3. Configure a backend. Set `METRICS_PUSHGATEWAY_URL` to your Pushgateway, such
   as `https://pushgateway.example.com`, and restart the instance.

4. Export once without waiting for the schedule:

   ```bash
   curl -X POST http://localhost:3001/api/admin/telemetry/exporters/pushgateway/export \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   { "status": "ok", "message": "export triggered for pushgateway" }
   ```

   Without a configured Pushgateway the answer is `404` with
   `exporter not found`.

5. In the admin console, open **Insight > Observability > Telemetry** to see
   each exporter's health.

## Choose a backend

| Name | Transport | What is sent |
|---|---|---|
| `pushgateway` | HTTP `PUT /metrics/job/<job>`, Prometheus text format, optional basic auth | Every metric, labels intact. Each push replaces the job's group. |
| `otlp` | OTLP over gRPC | Gauges as gauges, counters as cumulative sums, histograms as cumulative histograms, summaries as summaries. Labels become attributes. |
| `newrelic` | OTLP over HTTP, `api-key` header | The same as `otlp`, posted to the region's `/v1/metrics`. |
| `statsd` | DogStatsD over UDP | Gauges as `g`. Counters as `c`, with the change since the last export. A histogram as `.sum`, `.count` and one `.bucket` gauge per bound tagged `le:`. A summary as `.sum`, `.count` and one gauge per quantile tagged `quantile:`. Labels become tags. Packets are at most 8 KiB. |

A backend runs when its address or key is set. Each one runs on its own: one
that fails is marked unhealthy and turns healthy again on its next successful
export, without affecting the others.

## Reach a collector on your own network

Exporters refuse loopback, link-local and private addresses. A collector on
your own network, such as `10.0.4.12:4317`, needs its range in
`METRICS_ALLOWED_PRIVATE_NETWORKS`, for example `10.0.4.0/24`. Until then a
manual export answers `422` and names the setting.

## Send a tenant its own metrics

With `multitenant-customization`, a tenant admin saves one destination for
the tenant, and every export also pushes it the series labeled with that
tenant and nothing else, the same samples the tenant's scrape shows:

```bash
curl -X PUT http://localhost:3001/api/admin/telemetry/destination \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"kind": "otlp", "url": "https://otlp.example.com/v1/metrics", "headers": {"Authorization": "Bearer <collector key>"}}'
```

| Field | Meaning |
|---|---|
| `kind` | `otlp`, sent as OTLP over HTTP to the URL exactly as saved, or `pushgateway`, sent to `<url>/metrics/job/lyeve-core/tenant/<slug>`. |
| `url` | An `http` or `https` URL of at most 2,048 characters, with no credentials in it. Send those as a header. |
| `headers` | Up to 16 headers, 8 KiB in all. Left out, the stored headers stay while the URL keeps its scheme and host. `{}` sends none. |

- The URL and the headers are encrypted at rest. The answer, like `GET` on the
  same path, shows the `kind`, the `url`, the `header_names` alone and the
  `health` of the last push, beside `licensed`.
- A destination on a private or internal network is refused, and
  `METRICS_ALLOWED_PRIVATE_NETWORKS` never opens one for a tenant.
- A destination moved to another host must send its headers again, so a
  stored key never follows a URL somebody else chose.
- `DELETE` removes the destination and stays free. If the license lapses, the
  stored destination keeps receiving, and a new or changed one answers `402`.

## Check exporter health

`GET /api/admin/telemetry/exporters` lists each configured backend:

```json
{
  "data": [
    { "name": "pushgateway", "healthy": true, "last_export": "2026-10-01T09:40:00Z", "exports": 120, "failures": 0 }
  ],
  "limit": 50,
  "offset": 0,
  "total_count": 1
}
```

A backend that has failed also shows `last_failure`, `last_error` and
`last_error_class`. Addresses and URLs in the error text are redacted.

## Settings

Set these as environment variables or in the admin console. They are read when
the instance starts, so restart it after a change.

| Variable | What it does | Default |
|---|---|---|
| `METRICS_EXPORT_INTERVAL` | How often to export, such as `30s` or `1m`. `0` turns the schedule off and leaves manual export. | `30s` |
| `METRICS_ENABLED_EXPORTERS` | Comma-separated backends to run. Empty or `all` runs every backend that is configured. | `all` |
| `METRICS_PUSHGATEWAY_URL` | Pushgateway base URL. | unset |
| `METRICS_PUSHGATEWAY_JOB` | Pushgateway job name. Slashes and `..` are refused. | `lyeve-core` |
| `METRICS_PUSHGATEWAY_USERNAME`, `METRICS_PUSHGATEWAY_PASSWORD` | Pushgateway basic auth. | unset |
| `METRICS_OTLP_ENDPOINT` | OTLP gRPC endpoint as `host:port`. | unset |
| `METRICS_OTLP_INSECURE` | Connect to the OTLP endpoint without TLS. | `false` |
| `METRICS_OTLP_HEADERS` | Comma-separated `key=value` headers for OTLP. | unset |
| `METRICS_STATSD_ADDRESS` | DogStatsD address as `host:port`. | unset |
| `METRICS_STATSD_PREFIX` | Prefix for every StatsD metric name. | `lyeve` |
| `METRICS_STATSD_TAGS` | Comma-separated tags sent with every StatsD metric. | unset |
| `METRICS_NEWRELIC_API_KEY` | New Relic license key. | unset |
| `METRICS_NEWRELIC_ENDPOINT` | New Relic OTLP URL for your region. For the EU use `https://otlp.eu01.nr-data.net:4318`. | `https://otlp.nr-data.net:4318` |
| `METRICS_ALLOWED_PRIVATE_NETWORKS` | Comma-separated CIDR ranges or addresses an exporter may reach. | unset |

Metrics export runs on every install. If you set `LYEVE_PLUGINS` to choose
which features start, include `telemetry` in it. See
[licensing and tiers](../concepts/licensing-and-tiers.md).

:::caution[Beta]
Metrics export is in beta. Plan around these limits:

- Histograms keep the instance's own buckets for every backend.
- There is one set of exporters per instance. A tenant adds at most one
  destination of its own.
- The `tenant` label holds the tenant slug.
:::

## Routes

| Method | Path | Who | Purpose |
|---|---|---|---|
| `GET` | `/api/admin/telemetry/metrics` | Admin | Prometheus text format. A super admin gets every metric, a tenant admin only the samples labeled with their tenant. |
| `GET` | `/api/admin/telemetry/exporters` | Super admin | Health of each backend, with `limit` (default 50, at most 500) and `offset`. |
| `POST` | `/api/admin/telemetry/exporters/{name}/export` | Super admin | Export to one backend now. |
| `GET` | `/api/admin/telemetry/destination` | Admin | The tenant's own destination, or `null`, and `licensed`. |
| `PUT` | `/api/admin/telemetry/destination` | Admin | Save it: `kind`, `url`, `headers`. Needs `multitenant-customization`. |
| `DELETE` | `/api/admin/telemetry/destination` | Admin | Remove it. `204`. |

## Errors

| Status | Message | Cause |
|---|---|---|
| `402` | `payment_required`, naming `feature:multitenant_customization` | A new or changed tenant destination without the license. |
| `422` | `kind must be otlp or pushgateway`, `url must be an absolute http or https URL`, `url must not carry credentials, send them as a header`, `at most 16 headers` | The destination is not valid. |
| `422` | `the destination moved to another host, send its headers again`, code `telemetry.headers_required` | A new host with the headers left out. |
| `422` | `the destination is on a network this instance does not send to` | A private or internal address. |
| `422` | `the instance has no encryption key to seal the destination with` | Set `ENCRYPTION_KEY`. |
| `403` | `instance-wide metrics need super_admin` | The caller is not a super admin and the request named no tenant. |
| `404` | `exporter not found` | No backend by that name is configured. |
| `422` | `export target is on a private network the guard refuses; list it under metrics_allowed_private_networks` | Add the range to `METRICS_ALLOWED_PRIVATE_NETWORKS`. |
| `429` | `export cooldown active; minimum 5 seconds between manual exports` | Wait five seconds between manual exports to one backend. |
| `502` | `export target did not accept the push; see the exporter's health row` | The backend could not be reached or refused the data. |
| `503` | `metrics not initialized` | The instance is still starting. |

## Related

- [Operator guide](../guides/operator/index.md#watch-a-running-instance):
  scrape `/api/admin/metrics` with Prometheus.
- [Scale and tune](../scaling/index.md): trace sampling and the database pool.
- [Request profiling](request-profiling.md): which route is slow, and why.
- [Logs](logs.md): what one request did.
