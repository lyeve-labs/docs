---
title: Analytics
description: Collect daily usage figures per tenant, forward events to GA4, Mixpanel, Plausible, Segment and more with a delivery log, and let users export or erase their data.
sidebar:
  order: 3
---

> **Included free** on every install, with the GA4, Mixpanel, Plausible and Segment destinations and a delivery log on every event. Retries, replaying a failed delivery, and the PostHog, Amplitude and webhook destinations need a license with the `analytics-pro` feature. See [pricing](https://lyeve.com/pricing).

Analytics answers "how is this tenant using the product" and passes what
happens in your content on to the analytics service you already use. It keeps
one row of figures per tenant per day, forwards every content change as an
event to the services you connect, records whether each one arrived, and lets
a signed-in user download or erase their own data.

:::tip[Best for]
- **Reporting usage per tenant.** Daily active users, API calls, storage and
  new users, one row per tenant per day, with totals over any range and a CSV
  export. Computing a day needs the [audit log](audit-log.md).
- **Seeing content activity in the analytics tool your team already reads.**
  Every content create, update and delete is sent as an event to the services
  you connect.
- **Tracking product events of your own.** Send events such as
  `checkout_started` through the same connections.
- **Not losing events when a service is down.** With `analytics-pro`, a
  failed delivery is retried with a growing wait, and one that still fails
  waits in a queue for you to replay.
- **Self-service privacy requests.** A signed-in user downloads their data or
  asks for it to be erased, with no admin involved.
:::

:::note[Use something else when]
- You want request volume, errors and latency per endpoint.
  [API analytics](api-analytics.md) breaks those down by hour.
- You want what a tenant has used against its plan limits.
  [Usage and quotas](usage-and-quotas.md) tracks that.
:::

## How it works

| Part | What it gives you | Where |
|---|---|---|
| **Daily figures** | `dau`, `api_calls`, `storage` and `new_users` for one tenant and one day, computed when you ask. | Admin API |
| **Event forwarding** | Content events and your own events, saved and sent to each enabled service, with the outcome kept on each event. | Admin API |
| **Tenant usage** | A super admin's view of usage and activity across tenants, and each tenant's own view. | Admin and Content API |
| **Privacy requests** | A user's own export and erasure. | Content API |

The admin console has no page for these figures, so you work with them through
the API below.

## Try it

You need an admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md) shows how to get one.

1. Send an event of your own:

   ```bash
   curl -X POST http://localhost:3001/api/admin/analytics/events \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"event_name": "checkout_started", "event_category": "store", "properties": {"cart_value": 49}}'
   ```

   The answer is `201`:

   ```json
   { "id": "7c9fbec4-d3b2-4d06-aa63-2ee70f026940", "tenant_id": "default", "event_name": "checkout_started", "event_category": "store", "properties": { "cart_value": 49 }, "provider_types": null, "processed": false, "created_at": "2026-10-01T09:08:51Z" }
   ```

2. List the events in that category:

   ```bash
   curl "http://localhost:3001/api/admin/analytics/events?category=store&limit=10" \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer is `{"data": [...], "total_count": n, "limit": 10, "offset": 0}`.

3. Connect a service. It stays off until `enabled` is `true`:

   ```bash
   curl -X POST http://localhost:3001/api/admin/analytics/providers \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name": "main-ga4", "type": "ga4", "enabled": false, "config": {"measurement_id": "G-XXXXXXX", "api_secret": "<secret>"}}'
   ```

   The answer is `201` with the saved connection. Its `config` comes back as
   `{"redacted": true}`, so a secret is never returned. Saving a connection
   needs `ENCRYPTION_KEY`, because its secrets are encrypted at rest. Without
   it the answer is `503` with the code `analytics.secrets_unavailable`.

4. List your connections:

   ```bash
   curl http://localhost:3001/api/admin/analytics/providers \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer is `{"data": [...], "licensed": false}`. Each connection in
   `data` carries its `id`, `name`, `type`, `enabled`, the redacted `config`
   and its `retry_policy`. `licensed` says whether this install may use the
   paid options below.

5. Remove it again with `DELETE /api/admin/analytics/providers/{id}`, which
   answers `204`.

## Compute daily figures

Figures for a day exist once you compute them. Call the aggregate route for
the day you want, or once a day from your own scheduler:

```bash
curl -X POST http://localhost:3001/api/admin/analytics/aggregate \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"date": "2026-09-30"}'
```

```json
{ "id": "fc6d4bd7-32cc-4191-8517-b1e1ff72ac01", "tenant_id": "default", "date": "2026-09-30", "dau": 42, "api_calls": 18250, "storage": 73400320, "new_users": 3, "computed_at": "2026-10-01T00:05:12Z" }
```

With no body, the date is today in UTC. Running it again for the same day
replaces that day's row. A super admin can compute one day for every tenant
with `POST /api/admin/analytics/aggregate-all`.

:::caution
Daily active users are counted from the [audit log](audit-log.md), which needs
a license with the `audit` feature. Without it, computing a day answers `503`
with `aggregation failed`.
:::

Read the stored days back, newest first, or the totals over a range:

```bash
curl "http://localhost:3001/api/admin/analytics/daily?from=2026-09-01&to=2026-09-30&limit=30" \
  -H "Authorization: Bearer $TOKEN"

curl "http://localhost:3001/api/admin/analytics/summary?from=2026-09-01&to=2026-09-30" \
  -H "Authorization: Bearer $TOKEN"
```

```json
{ "days": 30, "total_dau": 1260, "total_api_calls": 547500, "total_storage": 73400320, "total_new_users": 41 }
```

`from` and `to` take `YYYY-MM-DD`. `GET /api/admin/analytics/export` downloads
the same rows as CSV, or as NDJSON with `format=ndjson`.

## Forward events to an analytics service

Each tenant connects its own services. Content events use the names
`after_create`, `after_update` and `after_delete`, with the content type as the
category. Your own events go to every enabled service, or only to the types
you list in `provider_types`, such as `["segment"]`.

| `type` | `config` keys | Free or `analytics-pro` |
|---|---|---|
| `ga4` | `measurement_id`, `api_secret`, `api_url`, `timeout_seconds` | Free |
| `mixpanel` | `project_token`, `api_url`, `timeout_seconds` | Free |
| `plausible` | `site_id`, `api_url`, `api_token`, `timeout_seconds` | Free |
| `segment` | `write_key`, `api_url`, `timeout_seconds` | Free |
| `posthog` | `api_key`, `api_url`, `timeout_seconds` | `analytics-pro` |
| `amplitude` | `api_key`, `api_url`, `timeout_seconds` | `analytics-pro` |
| `webhook` | `url`, `timeout_seconds` | `analytics-pro` |

- `timeout_seconds` defaults to `10`.
- GA4's `api_url` defaults to the global endpoint. Set
  `https://region1.google-analytics.com/mp/collect` to keep collection in the
  EU.
- PostHog defaults to `https://us.i.posthog.com` and Amplitude to
  `https://api2.amplitude.com/2/httpapi`. Point `api_url` at the EU endpoint or
  at your own PostHog. Both receive the event id as their deduplication key,
  so a retry that already arrived is dropped there.
- A `webhook` URL must be `https://`. Every address a connection sends to is
  checked when you save it and again on each delivery, and a private or
  internal one answers `422` with `the destination address is not allowed`.
- `PUT /api/admin/analytics/providers/{id}` changes `name`, `enabled`,
  `config` or `retry_policy`.
- The secrets in `config` are encrypted at rest and never returned. Saving a
  connection needs `ENCRYPTION_KEY`.

## Check that events arrived

Every event records its delivery on every install. `GET
/api/admin/analytics/events/{id}` shows `processed`, which turns `true` once
every service accepted the event, `processed_at`, and `error`, which names each
service still owed the event with its last failure.

`GET /api/admin/analytics/deliveries` lists the deliveries still owed, newest
first. Each carries the `event_id`, the `provider_id` and `provider_type`, its
`status` (`pending` while a retry waits, `failed` when none is left),
`attempts`, `last_error` and `next_attempt_at`. Filter with `?status=pending`
or `?status=failed`, and `limit` goes up to 200. A delivered event leaves no
row. A `failed` delivery is deleted 30 days after its last attempt, and the
event keeps its record.

Every install deletes tracked events 90 days after they were recorded, with
any delivery still owed for them. Set `ANALYTICS_EVENT_RETENTION_DAYS` to keep
them longer, or `0` to keep them forever. The daily figures are kept apart and
are not deleted.

## Retry and replay failed deliveries

Without `analytics-pro`, each event gets one attempt per service. With it, a
connection takes a retry policy:

```json
{"retry_policy": {"max_attempts": 5, "backoff_seconds": 30}}
```

`max_attempts` counts the first attempt and goes from 1 to 10. The first retry
waits `backoff_seconds`, from 1 to 3600, and each later one waits twice as long
as the one before, up to an hour. A delivery that runs out of attempts stays
in the list as `failed` until you replay it:

```bash
curl -X POST http://localhost:3001/api/admin/analytics/deliveries/$DELIVERY_ID/replay \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `{"delivered": true}`, or `false` with the delivery as it stands
after the new failure. A replay sends once, now.

Without `analytics-pro`, creating a PostHog, Amplitude or webhook connection,
setting a retry policy and replaying answer `402` naming
`feature:analytics-pro`. If the license lapses, every stored connection keeps
delivering and every stored policy keeps retrying. Clearing a policy, sending
back the stored one and every other edit stay free.

## Verify a webhook delivery

The answer that creates a `webhook` connection carries its `signing_secret`
once. Changing the URL, or sending `"rotate_secret": true` on an update,
answers a new one the same way. Each delivery is a JSON `POST` of `id`,
`tenant_id`, `event_name`, `event_category`, `properties` and `created_at`,
with these headers:

| Header | Value |
|---|---|
| `X-Lyeve-Timestamp` | Unix seconds when the delivery was sent. |
| `X-Lyeve-Signature` | `sha256=` and the hex HMAC-SHA256, keyed by the secret, of the timestamp, a `.` and the raw body. |
| `X-Lyeve-Event-Id` | The event id, the same on every attempt, so you can drop a duplicate. |

Verify the signature over the raw body, and refuse a timestamp more than a few
minutes old.

## Read usage across tenants

A super admin lists usage per tenant with `GET /api/admin/analytics/tenants`
and reads one tenant with `GET /api/admin/analytics/{slug}`. These figures are
computed in the background: `POST /api/admin/analytics/refresh` recomputes them
for every tenant and answers `202`.

A signed-in user reads their own tenant's figures on the Content API with
`GET /api/v1/analytics`. The answer carries `tenant_id`, plus `usage` and
`activity` once figures exist for that tenant. It allows 10 requests a minute
per tenant.

## Let users export or erase their data

A signed-in user calls these on the Content API for their own account:

```bash
curl -X POST http://localhost:3002/api/v1/gdpr/export \
  -H "Authorization: Bearer $TOKEN" -o my-data.ndjson
```

The download is one JSON object per line, each tagged with `data_type`:
`content`, `audit_log`, `media_metadata` or `user`.

`POST /api/v1/gdpr/erase` answers `202`:

```json
{ "erase_id": "5a0f9c2e-1b7d-4e3a-8c6f-2d9e0a4b7c15", "message": "GDPR erasure initiated. Your data will be removed shortly." }
```

:::danger
Erasure applies to the account that calls it, so never run it with your own
admin token. It runs in the background, soft-deletes the content the user
created, deletes the media they uploaded, and anonymizes the account: the email
becomes `erased+<user id>@localhost`, and the password and roles are cleared.
API keys belong to the tenant and are kept.
:::

## Settings

| Variable | What it does | Default |
|---|---|---|
| `ANALYTICS_TENANT_RATE_LIMIT` | Requests per minute per tenant on `GET /api/v1/analytics`. A missing, invalid or negative value uses the default. `0` lets one request through a minute. | `10` |
| `ANALYTICS_FAILED_DELIVERY_RETENTION_DAYS` | Days a `failed` delivery is kept after its last attempt. A value under `1`, or one that is not a number, uses the default. | `30` |
| `ANALYTICS_EVENT_RETENTION_DAYS` | Days a tracked event is kept. `0` keeps events forever. A negative value, or one that is not a number, uses the default. | `90` |
| `ENCRYPTION_KEY` | Encrypts the secrets of every connection. Without it, saving a connection answers `503`. | unset |

Analytics runs on every install. If you set `LYEVE_PLUGINS` to choose which
features start, include `analytics` in it. See
[licensing and tiers](../concepts/licensing-and-tiers.md).

## Routes

"Admin" means an admin or super admin.

<details>
<summary>Every analytics route</summary>

| Method | Path | Who | Result |
|---|---|---|---|
| `GET` | `/api/admin/analytics/daily` | Admin | Stored days. Query: `from`, `to`, `limit`. |
| `GET` | `/api/admin/analytics/daily/{date}` | Admin | One day, or `404`. |
| `GET` | `/api/admin/analytics/summary` | Admin | Totals. Query: `from`, `to`. |
| `POST` | `/api/admin/analytics/aggregate` | Admin | Computes one day for your tenant. `201`. |
| `POST` | `/api/admin/analytics/aggregate-all` | Super admin | Computes one day for every tenant: `{"date": "...", "aggregated": 12}`. |
| `GET` | `/api/admin/analytics/tenants` | Super admin | Usage per tenant. Query: `sort` (`slug`, `name`, `plan`, `enabled`), `order` (`asc`, `desc`), `limit` (50), `offset`. |
| `GET` | `/api/admin/analytics/{slug}` | Super admin | One tenant's usage and activity. |
| `POST` | `/api/admin/analytics/refresh` | Super admin | Recomputes per-tenant usage in the background. `202`. |
| `POST` | `/api/admin/analytics/{slug}/refresh` | Super admin | The same for one tenant. `202`. |
| `GET` | `/api/admin/analytics/export` | Admin | Download. Query: `format` (`csv` default, or `ndjson`), `from`, `to`. |
| `GET` | `/api/admin/analytics/providers` | Admin | Your connections, as `{"data": [...], "licensed": bool}`. |
| `POST` | `/api/admin/analytics/providers` | Admin | Connect a service. `201`. PostHog, Amplitude, webhook and a retry policy need `analytics-pro`. |
| `GET`, `PUT`, `DELETE` | `/api/admin/analytics/providers/{id}` | Admin | Read, change or remove one. |
| `GET`, `POST` | `/api/admin/analytics/events` | Admin | List or send events. List query: `category`, `event`, `limit`. |
| `GET` | `/api/admin/analytics/events/{id}` | Admin | One event, with its delivery outcome. |
| `GET` | `/api/admin/analytics/deliveries` | Admin | Deliveries still owed, as `{"data": [...], "licensed": bool}`. Query: `status` (`pending` or `failed`), `limit` (at most 200). |
| `POST` | `/api/admin/analytics/deliveries/{id}/replay` | Admin | Send one delivery again now. Needs `analytics-pro`. |
| `GET` | `/api/admin/analytics/dashboard` | Admin | `total_content`, `total_media`, `total_users`, `api_calls_24h`. Today the first three are always `0`, and `api_calls_24h` holds the daily active users computed for today. |
| `GET` | `/api/v1/analytics` | Signed in | Your tenant's usage and activity. |
| `POST` | `/api/v1/gdpr/export` | Signed in | Your data as NDJSON. |
| `POST` | `/api/v1/gdpr/erase` | Signed in | Erases your data. `202`. |

</details>

## Errors

| Status | Message |
|---|---|
| `400` | `invalid 'from' date, expected YYYY-MM-DD` (or `'to'`, or `invalid date, expected YYYY-MM-DD`) |
| `400` | `type must be one of: ga4, mixpanel, plausible, segment, posthog, amplitude, webhook` |
| `400` | `name is required`, `name contains invalid characters` |
| `400` | `event_name is required` |
| `400` | `format must be 'csv' or 'ndjson'` |
| `400` | `status must be pending or failed`, `invalid delivery ID` |
| `402` | `payment_required`, naming `feature:analytics-pro` |
| `403` | `super_admin required` |
| `404` | `daily metric not found`, `provider not found`, `event not found`, `tenant not found`, `delivery not found` |
| `422` | `retry_policy.max_attempts must be between 1 and 10`, `retry_policy.backoff_seconds must be between 1 and 3600 when max_attempts is above 1` |
| `422` | `config.url must be an absolute https:// URL of at most 2048 characters`, `the destination address is not allowed` |
| `429` | `rate limit exceeded (<n> req/min per tenant)` on `GET /api/v1/analytics` |
| `503` | `aggregation failed` |
| `503` | `provider secrets cannot be stored right now`, code `analytics.secrets_unavailable`, when the instance has no `ENCRYPTION_KEY` |

## Related

- [API analytics](api-analytics.md): requests, errors and latency per endpoint.
- [Usage and quotas](usage-and-quotas.md): usage against plan limits.
- [Audit log](audit-log.md): the record daily active users are counted from.
