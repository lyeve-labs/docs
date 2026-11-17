---
title: "25. Captures, synthetic checks and telemetry"
description: "Look at captured requests, choose what is captured, replay a saved set and compare the answers, check the site on a schedule, browse the metrics and send them to a collector."
sidebar:
  order: 25
covers:
  console:
    - "/admin/observability/captures"
    - "/admin/observability/captures/rules"
    - "/admin/observability/captures/sets"
    - "/admin/observability/synthetic"
    - "/admin/observability/telemetry"
    - "/admin/observability/telemetry/export"
  api:
    - "GET /api/admin/request-captures"
    - "POST /api/admin/request-captures/rules"
    - "POST /api/admin/request-captures/sets"
    - "POST /api/admin/request-captures/sets/{id}/replay"
    - "POST /api/admin/synthetic-monitoring/probes"
    - "POST /api/admin/synthetic-monitoring/probes/{id}/run"
    - "PUT /api/admin/synthetic-monitoring/probes/{id}/alert-channels"
    - "GET /api/admin/telemetry/metrics"
    - "PUT /api/admin/telemetry/destination"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson you keep a record of real requests and replay them after a change, check the
site from outside on a schedule, and send the engine's numbers to the tools you already watch.
You need an admin account, a super admin for the engine-wide views, and a token in `TOKEN` for
the API steps. [Lesson 24](performance.md) is the one before this.

## Look at captured requests

> **Included free** on every install, with every request kept for 24 hours.

<div class="paths">
<div data-path="console">

Open **Insight > Observability > Request capture**. The **Captures** tab lists each request
with its method and path, its **Status**, how long it **Took** and when it was **Captured**,
newest first. A super admin sees every tenant's requests, with a **Tenant** column.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/request-captures?method=GET&limit=5" \
  -H "Authorization: Bearer $TOKEN"
```

Copy the `id` of a read of `/api/v1/content/trip` into `CAPTURE_ID` for the replay below.

</div>
</div>

Headers that carry credentials are stored redacted.
[Replay a call safely](../features/request-capture.md#replay-a-call-safely) replays one capture
and shows the diff.

## Choose what is captured

> **Requires** a license with the `request-capture-pro` feature. See [pricing](https://lyeve.com/pricing/).

Without rules, every request is captured and kept for a day. Once a rule is on, only matching
requests are kept, for as long as the rule says.

<div class="paths">
<div data-path="console">

1. On **Request capture**, choose the **Capture rules** tab, then **New rule**.
2. Enter `/api/v1/content/trip/**` in **Route** and pick `POST` in **Method**.
3. Enter `100` in **Sample, as a percentage** and pick `7 days` in **Kept for**. Keep
   **Apply this rule** on.
4. Choose **Create**. The rule reads **Capturing**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/request-captures/rules \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"route_pattern": "/api/v1/content/trip/**", "method": "POST", "sample_rate": 1, "retention_seconds": 604800}'
```

</div>
</div>

On a free install the tabs show **Capture rules and replay sets are not included**, and
**New rule** and **New replay set** are turned off.

![The Capture rules tab on a free install, with the not included notice and New rule turned off.](./media/captures-and-telemetry-rules-free.png)

![The Capture rules tab on a licensed install, with a rule for trip writes kept seven days.](./media/captures-and-telemetry-rules-licensed.png)

## Replay a set and compare the answers

> **Requires** a license with the `request-capture-pro` feature. See [pricing](https://lyeve.com/pricing/).

A replay set is a named list of captures that you run again after a change, to see whether the
answers moved.

<div class="paths">
<div data-path="console">

1. On the **Captures** tab, tick the box on two or three reads of `/api/v1/content/trip`.
2. Choose **New replay set from 3**, the button names how many you ticked. Enter `trip reads`
   in **Name** and choose **Create**.
3. On the **Replay sets** tab, choose **Replay** on the set's row and confirm.

The page lists each call with **Same** or **Changed**, and what changed in the headers or the
body. A set that holds a write warns you first and offers **Replay including writes**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/request-captures/sets \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "trip reads", "capture_ids": ["'"$CAPTURE_ID"'"]}'
```

Copy the set's `id` into `SET_ID`, then replay it:

```bash
curl -X POST http://localhost:3001/api/admin/request-captures/sets/$SET_ID/replay \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"confirm_replay": true}'
```

</div>
</div>

![Ticking three trip reads, saving them as a replay set, replaying it and reading Same on each call.](./media/captures-and-telemetry-replay-set.webm)

A set takes only captures from the tenant you are signed in to, up to 50 of them.

## Check the site on a schedule

> **Included free** on every install, with three probes per tenant, each run at most every 300 seconds. More probes and shorter intervals need a license with the `synthetic-monitoring-pro` feature. See [pricing](https://lyeve.com/pricing/).

A probe checks an address from the engine on a schedule and raises an alert after repeated
failures.

<div class="paths">
<div data-path="console">

1. Open **Insight > Observability > Synthetic monitoring** and choose **New probe**.
2. Enter `Travel site` in **Name** and pick **Health check** under **What it checks**.
3. Enter your public site's address in **URL** and `200` in **Expected status**.
4. Keep `300` in **Interval, in seconds** and `3` in **Failures before an alert**, keep
   **Run on the schedule** on, and choose **Create**.
5. Choose **Run now** on the probe's row instead of waiting. **Last run** updates.

**Notice channels** on the row sets where down and recovery notices go. **Unacknowledged
alerts** above the list shows a probe that is down, with **Acknowledge**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/synthetic-monitoring/probes \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Travel site", "type": "health_check", "config": {"url": "https://example.com/", "expected_status": 200}, "regions": ["eu-west"], "alert_threshold": 3}'
```

Copy the `id` into `PROBE_ID`, run it now and send its notices by email:

```bash
curl -X POST http://localhost:3001/api/admin/synthetic-monitoring/probes/$PROBE_ID/run \
  -H "Authorization: Bearer $TOKEN"

curl -X PUT http://localhost:3001/api/admin/synthetic-monitoring/probes/$PROBE_ID/alert-channels \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": ["ops@example.com"]}'
```

</div>
</div>

![Creating a health check probe for the travel site, running it now and the last run updating.](./media/captures-and-telemetry-new-probe.webm)

A probe can also create and delete a record through the API, sign in, or deliver a webhook.
[Set up a probe](../features/synthetic-monitoring.md#set-up-a-probe) describes each kind.

## Browse the metrics

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Open **Insight > Observability > Telemetry**. The **Metrics** tab lists each metric family with
its series and values. Type `lyeve_requests_total` in the filter to see the requests by status
class, method and tenant, and pick a **Type** to narrow the list. A tenant admin sees only the
series of their own tenant.

</div>
<div data-path="api">

```bash
curl -s http://localhost:3001/api/admin/telemetry/metrics \
  -H "Authorization: Bearer $TOKEN" | grep '^lyeve_requests_total'
```

</div>
</div>

A scraper reads the same series with `METRICS_TOKEN`, as
[Metrics export](../features/metrics-export.md) shows.

## Send a tenant's metrics to a collector

> **Requires** a license with the `multitenant-customization` feature. See [pricing](https://lyeve.com/pricing/).

<div class="paths">
<div data-path="console">

1. On **Telemetry**, choose the **Export** tab.
2. Under **Tenant destination**, pick **OTLP** in **Protocol** and enter your collector's
   address in **URL**.
3. Choose **Add header**, then enter `Authorization` in **Header** and the collector's key in
   **Value**.
4. Choose **Save**. After the next export the section shows whether the destination is healthy.

A super admin also sees **Exporters**, the engine-wide exporters, each with **Export now**.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/telemetry/destination \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"kind": "otlp", "url": "https://otlp.example.com/v1/metrics", "headers": {"Authorization": "Bearer <collector key>"}}'
```

</div>
</div>

On a free install the destination form is turned off and reads **Not enabled on this engine.**
under its **Save** button.

![The Export tab on a free install, with the tenant destination form turned off.](./media/captures-and-telemetry-export-free.png)

![The Export tab on a licensed install, with an OTLP destination and a stored header.](./media/captures-and-telemetry-export-licensed.png)

## Next

[26. Customize the console](customize-the-console.md) makes the console your team's own. For
depth, read [Request capture](../features/request-capture.md),
[Synthetic monitoring](../features/synthetic-monitoring.md) and
[Metrics export](../features/metrics-export.md).
