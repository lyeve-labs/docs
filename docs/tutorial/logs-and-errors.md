---
title: "23. Logs, errors and alerts"
description: "Find one request in the log, follow the live log, get an alert when log volume climbs, triage a recorded failure and choose where error spike alerts go."
sidebar:
  order: 23
covers:
  console:
    - "/admin/logs"
    - "/admin/logs/alerts"
    - "/admin/observability/log-tail"
    - "/admin/observability/errors"
    - "/admin/observability/errors/alerts"
  api:
    - "GET /api/admin/logs/search"
    - "GET /api/admin/logs/stream"
    - "PUT /api/admin/logging/alerts"
    - "POST /api/admin/error-tracking/codes"
    - "POST /api/admin/error-tracking/events/stack"
    - "GET /api/admin/error-tracking/alerts"
    - "POST /api/admin/error-tracking/alerts/{id}/resolve"
    - "PUT /api/admin/error-tracking/alerts/{id}/assignee"
    - "PUT /api/admin/error-tracking/alert-settings"
verified:
  engine: v0.62.0
  console: v0.28.0
---

> **Included free** on every install, with alerts by email. Slack, Discord, PagerDuty and HTTPS webhook alerts, and a tenant's own error spike threshold, need a license with the `alerts-pro` feature. See [pricing](https://lyeve.com/pricing/).

In this lesson you trace one request through the log, watch the log as it is written, and set
up the alerts that tell you something is wrong before a reader does. You need a super admin
account and a token in `TOKEN` for the API steps. [Lesson 22](tenants-and-costs.md) is the one
before this.

## Find one request in the log

Every request is logged with a request ID. Send one of your own with a read of the trips:

```bash
curl -s -o /dev/null http://localhost:3002/api/v1/content/trip \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Request-ID: tutorial-trips-1"
```

<div class="paths">
<div data-path="console">

1. Open **Insight > Logs**.
2. Under **Entries**, type `tutorial-trips-1` in the search box. The list narrows as you type to
   one **INFO** entry that reads `request`.
3. Pick a level in the box beside it, such as **ERROR**, to see only that level, or choose a
   level under **Volume and levels**.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/logs/search?request_id=tutorial-trips-1" \
  -H "Authorization: Bearer $TOKEN"
```

The entry's `fields` hold the method, the route, the status and how long it took. A request
reaches the search a few seconds after it is answered, so search again if `results` is empty.

</div>
</div>

**Volume and levels** at the top shows the entries of the last 24 hours, the **Default level**
and the **Sinks** the log is shipped to. [Find a line](../features/logs.md#find-a-line) lists
every filter.

![The Logs page searched for the request ID tutorial-trips-1, with one INFO entry.](./media/logs-and-errors-search.png)

## Follow the live log

<div class="paths">
<div data-path="console">

1. Open **Insight > Observability** and choose the **Live log tail** tab. The badge reads
   **Connected**.
2. Pick the minimum level, and type a word in **Keyword, then Enter** to keep only lines that
   hold it.
3. Read a trip in another tab. Its line appears at once.

**Pause** holds the list while new lines keep arriving, and **Clear** empties it.

</div>
<div data-path="api">

Start with what is buffered, and press Ctrl+C to stop:

```bash
curl -N "http://localhost:3001/api/admin/logs/stream?min_level=INFO&from_head=true" \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![The live log tail connecting, filtering on a keyword, and a new line arriving when a trip is read.](./media/logs-and-errors-live-tail.mp4)

## Get an alert when log volume climbs

A super admin writes volume rules. Each rule watches one level, or all of them, over a window,
and is checked every minute.

<div class="paths">
<div data-path="console">

1. On **Insight > Logs**, choose **Volume alerts**, then **New rule**.
2. Keep `ERROR` in **Level**, enter `100` in **Ceiling** and `5m` in **Window**. Leave
   **Tenant** and **Plugin** empty to count every entry, and keep **On**.
3. Under **Channels**, enter your address in **Email** and choose **Create**.

**Cooldown** below the list sets the quiet time after a rule fires, such as `10m`, with its own
**Save**.

</div>
<div data-path="api">

`PUT` replaces every rule at once:

```bash
curl -X PUT http://localhost:3001/api/admin/logging/alerts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"thresholds": [{"level": "ERROR", "max_count": 100, "window": "5m", "enabled": true, "channels": {"email": ["ops@example.com"]}}], "cooldown": "10m"}'
```

</div>
</div>

![Opening New rule on Volume alerts, setting the level, ceiling and window, adding an email channel and creating the rule.](./media/logs-and-errors-volume-rule.mp4)

## Triage a failure

Error tracking groups recorded failures by where they came from. To have one to look at,
register a code and send an event. Both are API calls:

```bash
curl -X POST http://localhost:3001/api/admin/error-tracking/codes \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"code": "TRIP_IMPORT_FAILED", "title": "Trip import failed", "severity": "ERROR", "category": "network"}'
```

Copy the `id` into `CODE_ID`, then send the event:

```bash
curl -X POST http://localhost:3001/api/admin/error-tracking/events/stack \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"error_code_id": "'"$CODE_ID"'", "endpoint": "/imports/trips", "message": "Feed timed out", "http_status": 504, "stack_trace": "main.importTrips()\n\t/app/import/trips.go:42"}'
```

<div class="paths">
<div data-path="console">

1. Open **Insight > Observability > Errors** and choose **Open** above **Failures**. The row
   reads **Trip import failed** with `Feed timed out`.
2. Choose **Assign** on the row, enter a name or an address in **Assignee** and choose
   **Save**.
3. Once the cause is fixed, choose **Resolve**. The failure moves to **Resolved**.

**Acknowledge** marks a failure as seen, and **Ignore** keeps it out of **Open** when it comes
back. **Recent events** below lists the latest events with their endpoint and status.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/error-tracking/alerts?status=open" \
  -H "Authorization: Bearer $TOKEN"
```

Copy the alert's `id` into `ALERT_ID`, then assign it and resolve it:

```bash
curl -X PUT http://localhost:3001/api/admin/error-tracking/alerts/$ALERT_ID/assignee \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"assignee": "ops@example.com"}'

curl -X POST http://localhost:3001/api/admin/error-tracking/alerts/$ALERT_ID/resolve \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![Filtering Failures to Open, assigning the trip import failure and resolving it.](./media/logs-and-errors-triage.mp4)

The next matching event opens a resolved failure again.

## Choose where spike alerts go

A spike is a sudden rise in a tenant's errors over the windows before it. Each tenant chooses
where its spike alerts go.

<div class="paths">
<div data-path="console">

1. On **Errors**, choose **Spike alerts**.
2. Under **Where alerts go**, enter your address in **Email** and choose **Save**. The page says
   **Spike alerts saved.**
3. With `alerts-pro`, pick **Custom** in **Threshold** under **What counts as a spike**, and set
   **Minimum errors in a window** and **Z-score**.

A new **HTTPS webhook URL** opens a **Webhook signing secret** dialog once, with the secret to
copy.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/error-tracking/alert-settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"channels": {"email": ["ops@example.com"]}}'
```

With `alerts-pro`, add `"spike_min_count": 20` and `"spike_z_score": 3.5` to set a threshold
of the tenant's own.

</div>
</div>

On a free install the Slack, Discord, HTTPS webhook and PagerDuty fields say they are not
enabled, and the tenant uses the default threshold.

![Spike alerts on a free install, with an email address and the other channels not enabled.](./media/logs-and-errors-spike-alerts-free.png)

![Spike alerts on an install licensed for alerts-pro, with every channel and a custom threshold.](./media/logs-and-errors-spike-alerts-licensed.png)

## Next

[24. Performance](performance.md) finds what is slow. For depth, read
[Logs](../features/logs.md), [Error tracking](../features/error-tracking.md) and
[Alerts](../features/alerts.md).
