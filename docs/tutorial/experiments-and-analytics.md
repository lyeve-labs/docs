---
title: "14. Experiments, analytics and recommendations"
description: "Test two layouts of a trip page, read how the API is used, send tracked events to an analytics service, and see what readers are recommended."
sidebar:
  order: 14
covers:
  console:
    - "/admin/experiments"
    - "/admin/experiments/[id]"
    - "/admin/analytics"
    - "/admin/analytics-destinations"
    - "/admin/settings/recommendations"
  api:
    - "POST /api/admin/ab/experiments"
    - "POST /api/admin/ab/experiments/{id}/variants"
    - "POST /api/admin/ab/experiments/{id}/metrics"
    - "POST /api/admin/ab/experiments/{id}/start"
    - "POST /api/v1/ab/expose"
    - "POST /api/v1/ab/convert"
    - "GET /api/admin/ab/experiments/{id}/results"
    - "GET /api/admin/apianalytics/metrics/summary"
    - "POST /api/admin/analytics/providers"
    - "POST /api/v1/recommendations/behavior"
    - "POST /api/admin/recommendations/refresh"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson you split the readers of a trip page between two layouts and see which one
leads more of them to save a trip. Then you read how your API is used, forward events to an
analytics service, and look at what readers are recommended. You need published trips from
lesson 4 and an `admin` or `super_admin` token in `TOKEN`, as in
[lesson 13](flows.md).

## Create an experiment on a trip page

> **Requires** a license with the `ab-testing` feature.

<div class="paths">
<div data-path="console">

1. Open **Delivery > Experiments** and choose **New experiment**.
2. Enter `trip-page-layout` in **Name**, and the question it answers in
   **What it is testing**.
3. Keep `0.95` in **Confidence level**, enter `500` in **Minimum exposures**, and turn on
   **Stop automatically once significant**.
4. Choose **Create**. The list shows it as a draft. Open it from there.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/ab/experiments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "trip-page-layout", "min_sample_size": 500, "auto_stop_enabled": true}'
```

The answer is `201` with `"status": "draft"`. Save its `id` as `EXP`.

</div>
</div>

On an install without the feature, **Experiments** shows a card that says A/B testing is not
enabled on this engine.

![The Experiments page on a free install, with the not enabled card.](./media/experiments-and-analytics-experiments-free.png)

![The Experiments page on a licensed install, with trip-page-layout as a draft.](./media/experiments-and-analytics-experiments-licensed.png)

## Add the layouts and what counts as a win

<div class="paths">
<div data-path="console">

1. On the experiment's page, choose **New variant**. Enter `map-first` in **Name**, `50` in
   **Share of traffic, as a percentage** and `{"layout":"map"}` in
   **Config the application reads**. Tick **This is the control** and choose **Create**.
2. Add `photos-first` the same way, with `{"layout":"photos"}` and without the tick.
3. Choose **New metric**. Enter `Saved a trip` in **Name** and `trip_saved` in
   **Event name**, keep **conversion** as the **Type**, and choose **Create**.
4. Go back to **Experiments** and choose **Start** on the row.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/ab/experiments/$EXP/variants \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "map-first", "traffic_percentage": 50, "is_control": true, "config": "{\"layout\":\"map\"}"}'

curl -X POST http://localhost:3001/api/admin/ab/experiments/$EXP/variants \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "photos-first", "traffic_percentage": 50, "config": "{\"layout\":\"photos\"}"}'

curl -X POST http://localhost:3001/api/admin/ab/experiments/$EXP/metrics \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Saved a trip", "event_name": "trip_saved", "metric_type": "conversion"}'

curl -X POST http://localhost:3001/api/admin/ab/experiments/$EXP/start \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![Adding the two variants and the metric on the experiment page, then starting it from the list.](./media/experiments-and-analytics-variants.webm)

The **Event name** has to match what your application sends exactly. A mismatch counts
nothing and shows no error. When the shares do not add up to 100, the page says so.

## Show a layout and count a conversion

Your trip page does this with the reader's own token, so it is API only. Ask which layout
this reader sees:

```bash
curl -X POST http://localhost:3002/api/v1/ab/expose \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"experiment_id\": \"$EXP\"}"
```

The answer names the `variant`, and your page draws that layout. When the reader saves a
trip, report it:

```bash
curl -X POST http://localhost:3002/api/v1/ab/convert \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"experiment_id\": \"$EXP\", \"event_name\": \"trip_saved\"}"
```

## Read the results

<div class="paths">
<div data-path="console">

Open the experiment. **Results** lists each variant with how many readers were **Exposed**
and **Converted**, its **Rate** and its **Confidence**. Below **Minimum exposures**, no result
counts as significant, whatever the p-value says.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/ab/experiments/$EXP/results \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

A stopped experiment cannot be started again, so the console asks before **Stop**. Pause it
instead if you mean to come back to it.

## Read how your API is used

> **Requires** a license with the `apianalytics` feature.

<div class="paths">
<div data-path="console">

Open **Insight > API analytics**. It always shows the last 24 hours.

| Part | What it shows |
|---|---|
| The top row | **Total requests**, **Server errors**, **Client errors**, and the **P50 latency**, **P95 latency** and **P99 latency** |
| **Responses** | Requests split into success, client errors and server errors |
| **Where the errors are** | The paths that return errors, when there are any |
| **Top endpoints** | Paths by request count, with the error rate, the slowest p99 and the average size |
| **HTTP methods** and **Top user agents** | Requests grouped by method and by client |
| **Anomalies** | Hours whose volume, latency or error rate left the expected range |

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/apianalytics/metrics/summary \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![The API analytics page after a few hundred trip reads.](./media/experiments-and-analytics-api-analytics.png)

## Forward events to an analytics service

> **Included free** on every install, with the GA4, Mixpanel, Plausible and Segment
> destinations. Retries, sending a delivery again, and the PostHog, Amplitude and webhook
> destinations need a license with the `analytics-pro` feature.

<div class="paths">
<div data-path="console">

1. Open **Insight > Analytics destinations** and choose **New destination**.
2. Enter `main-ga4` in **Name** and pick the type in **Type**.
3. Under **Settings**, fill in the service's keys. Saved settings are never shown again.
4. Leave **On** on and choose **Create**.

**Deliveries owed** lists the events a destination refused, with the attempts, the next
attempt and the last error. **Send again** retries one.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/analytics/providers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "main-ga4", "type": "ga4", "enabled": true, "config": {"measurement_id": "G-XXXXXXX", "api_secret": "<secret>"}}'
```

The answer's `config` reads `{"redacted": true}`, because a secret is never returned.

</div>
</div>

![Opening New destination, picking Google Analytics 4, filling in its settings and creating it.](./media/experiments-and-analytics-destination.webm)

Saving a destination needs `ENCRYPTION_KEY` on the engine, because its secrets are encrypted
at rest.

## See what readers are recommended

> **Requires** a license with the `recommendations` feature.

Your application records what each reader does with a trip. Recording is API only, because
it runs with the reader's own token. Put a trip's id in `TRIP`:

```bash
curl -X POST http://localhost:3002/api/v1/recommendations/behavior \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"content_id\": \"$TRIP\", \"action\": \"like\"}"
```

The lists are not live. They change only after a super admin recomputes them, and only a
super admin can generate a feed for a named reader.

<div class="paths">
<div data-path="console">

1. Open **Settings > Recommendations** and choose **Recompute**. The page reports how long it
   took and what it computed.
2. **Trending now** lists the entries with the most activity this week.
3. Under **What one reader would see**, search a reader by email in **Find a reader** and
   choose **Generate**. The list shows each entry with why it was picked and its score.

**The split** shows how many readers get the recommended feed and how many a random
baseline, so you can see whether recommending changes what people read.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/recommendations/refresh \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![Recomputing, then generating one reader's feed from their email address.](./media/experiments-and-analytics-recommendations.webm)

## Next

[15. Connected editors](connected-editors.md) connects the builder to your instance. For
depth, read [A/B testing](../features/ab-testing.md),
[API analytics](../features/api-analytics.md), [Analytics](../features/analytics.md) and
[Recommendations](../features/recommendations.md).
