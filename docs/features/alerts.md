---
title: Alerts
description: "Get an email, a chat message, a PagerDuty incident or a signed webhook call when a job fails, errors spike, log volume jumps or a probe goes down."
sidebar:
  order: 20
capability_of: cron
---

> **Included free** on every install, with alerts by email. Slack, Discord, PagerDuty and signed webhook channels, a tenant's own error spike threshold and error events older than 30 days need a license with the `alerts-pro` feature. See [pricing](https://lyeve.com/pricing).

Four features watch your instance and can tell people when something breaks: a
[scheduled job](scheduled-jobs.md) that keeps failing, an
[error](error-tracking.md) rate that spikes, a [log](logs.md) volume that
passes a limit, and an [uptime probe](synthetic-monitoring.md) that goes down.
Each one sends its alert to the channels you set on it. Email works on every
install. With `alerts-pro`, the same alert also reaches Slack, Discord,
PagerDuty or a webhook of yours that can check the alert came from your
instance.

:::caution[Beta]
The paid channels are in beta. They work as this page describes, and their
fields can still change before they are marked stable.
:::

:::tip[Best for]
- **A nightly export nobody watches.** Email the team when it fails three
  nights running, and again when it works.
- **An on-call rotation.** A failing job or a probe that goes down opens a
  PagerDuty incident, and the recovery resolves it.
- **The channel your team already reads.** Post error spikes and log floods
  to Slack or Discord.
- **Opening a ticket automatically.** Send a signed webhook to your incident
  tool, which can verify that it came from your instance.
:::

:::note[Use something else when]
- You want to react with logic of your own, such as retrying in another way.
  A [flow](flows.md) can start on the `cron.job.failed` event on every
  install.
- You want a message when content changes, not when something breaks.
  [Webhooks](webhooks.md) do that.
:::

## How it works

| What is watched | When it alerts | Where you set the channels | Who sets them |
|---|---|---|---|
| A scheduled job | After a number of failed runs in a row that you choose, from 1 to 100. The first run that works sends a recovery notice. | The job's `alerts` object | The job's tenant admin |
| Error spikes | When the tenant's error count over the last five minutes stands well above the six windows before it. At most one alert every 15 minutes per tenant. | `PUT /api/admin/error-tracking/alert-settings` | The tenant's admin |
| Log volume | When the entries a rule watches pass its `max_count` within its window. At most once per `cooldown`. | Each rule's `channels` | A super admin |
| An uptime probe | When the probe raises an alert after repeated failures. The next passing run sends a recovery notice. | `PUT /api/admin/synthetic-monitoring/probes/{id}/alert-channels` | The probe's tenant admin |

Every alert also stays where its feature keeps it: the job's history, the
error alert list, the live log tail and the probe's alert list. Those need no
channel and no license.

| Channel | Field | Free or `alerts-pro` | What it receives |
|---|---|---|---|
| Email | `email` | Free | A message to up to 10 addresses, through the instance's [email](email.md). |
| Slack | `slack_url` | `alerts-pro` | `{"text": "..."}` on a Slack incoming webhook. |
| Discord | `discord_url` | `alerts-pro` | `{"content": "..."}` on a Discord webhook. |
| PagerDuty | `pagerduty_routing_key` | `alerts-pro` | An event on the PagerDuty Events API v2. |
| Your webhook | `webhook_url` | `alerts-pro` | A signed JSON body, below. |

Every surface takes the same channel fields, so a setting you write for one
works on the others.

## Try it

You need an admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md) shows how to get one. Email
needs a working [email](email.md) transport. Without one, the email channel
is skipped and everything else still works.

1. Create a job that fails on every run and emails you after the second
   failure:

   ```bash
   curl -X POST http://localhost:3001/api/admin/cron/jobs \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "name": "always-fails",
           "schedule": "0 3 * * *",
           "endpoint": "https://httpbin.org/status/500",
           "alerts": {"email": ["you@example.com"], "failure_threshold": 2}
         }'
   ```

   The answer is `201`. Copy its `id` into `JOB_ID`.

2. Run the job now, twice, a few seconds apart:

   ```bash
   curl -X POST http://localhost:3001/api/admin/cron/jobs/$JOB_ID/trigger \
     -H "Authorization: Bearer $TOKEN"
   ```

   Each answer is `202`. After the second run, `consecutive_failures` on
   `GET /api/admin/cron/jobs/$JOB_ID` is `2`, and the email arrives with the
   subject `Cron job "always-fails" failed 2 runs in a row. The endpoint answered 500.`

3. Ask whether this install may add a paid channel:

   ```bash
   curl http://localhost:3001/api/admin/error-tracking/alert-settings \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   {"channels": {}, "spike_min_count": null, "spike_z_score": null, "defaults": {"spike_min_count": 10, "spike_z_score": 2.5}, "licensed": false, "free_event_window_days": 30}
   ```

   `licensed` is `false` on an install without `alerts-pro`. Adding a Slack
   URL to the job now answers `402` naming `feature:alerts-pro`, and the job
   keeps its email alert.

4. Delete the job when you are done. The answer is `204`.

## Alert on a failing job

Send `alerts` in the body of any route that creates or updates a job:
`POST` and `PUT` on `/api/admin/cron/jobs` and on `/api/admin/jobs`.

```json
{
  "alerts": {
    "email": ["ops@example.com"],
    "slack_url": "https://hooks.slack.com/services/T000/B000/XXXX",
    "pagerduty_routing_key": "R0123456789abcdef0123456789abcde",
    "failure_threshold": 3
  }
}
```

| Field | Meaning | Default |
|---|---|---|
| `email`, `slack_url`, `discord_url`, `webhook_url`, `pagerduty_routing_key` | The channels, described above. Any one is enough. | none |
| `failure_threshold` | Failed runs in a row before the alert, from 1 to 100. | `1` |

- A run fails when the endpoint answers `400` or above, or cannot be reached.
- Each channel hears once per streak of failures, not once per failed run,
  and the first run that works sends the recovery notice to the same channels.
- An update that leaves `alerts` out keeps the job's current setting.
  `"alerts": null`, or an object naming no channel, removes them.
- The admin console's **Operations > Scheduled jobs** has the email, Slack,
  Discord and webhook fields in each job's form. Set PagerDuty over the API.

The line a chat channel receives reads
`Cron job "nightly-export" failed 3 runs in a row. The endpoint answered 500.`,
or `Cron job "nightly-export" recovered after 3 failed runs in a row.`

## Alert on an error spike

Each tenant sets where its spike alerts go:

```bash
curl -X PUT http://localhost:3001/api/admin/error-tracking/alert-settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"channels": {"email": ["ops@example.com"]}}'
```

The answer is the setting, as `GET` on the same path returns it:
`channels`, the tenant's own thresholds (`null` for the default), the
`defaults`, `licensed` and `free_event_window_days`.

| Field | Meaning | Free or `alerts-pro` |
|---|---|---|
| `channels` | The channels, described above. | Email free |
| `spike_min_count` | The fewest errors in a window that can count as a spike, from 1 to 1,000,000. Default `10`. | `alerts-pro` |
| `spike_z_score` | How many standard deviations above the earlier windows a count must stand, from 0.5 to 20. Default `2.5`. | `alerts-pro` |

- A field left out keeps what is stored, and `null` clears it. A cleared
  threshold goes back to the default.
- The check runs every minute and compares each tenant's errors on their own.
- The alert names the tenant, the count, the window and how far above the
  baseline it stands.

An operator can also send every tenant's spikes to one place with
`SLACK_WEBHOOK_URL` and `DISCORD_WEBHOOK_URL`, described on
[error tracking](error-tracking.md#settings). Those belong to the instance's
configuration and send on every install.

## Alert on log volume

A super admin writes the volume rules, and each rule carries its own
channels:

```bash
curl -X PUT http://localhost:3001/api/admin/logging/alerts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "thresholds": [{
          "id": "errors",
          "level": "ERROR",
          "max_count": 100,
          "window": "5m",
          "enabled": true,
          "channels": {"email": ["ops@example.com"]}
        }],
        "cooldown": "10m"
      }'
```

- The rules are checked every minute in the background, whether or not
  anyone reads the volume.
- A rule keeps its `id` across writes, and a rule sent without one gets one.
- A rule with a `tenant_id` counts only that tenant's entries. One without
  counts every tenant's.
- Every firing also reaches the live log tail as an `ALERT` entry.

`POST /api/admin/logging/alerts` adds a single rule, and
`GET`, `PUT` and `DELETE /api/admin/logging/alerts/{id}` handle one rule at a time.
`GET /api/admin/logging/alerts` answers `licensed` beside the rules. A super
admin reads every rule. A tenant admin reads only the rules scoped to their
own tenant, with each email address shortened. [Logs](logs.md#watch-log-volume)
covers the rule fields.

## Alert when a probe goes down

Each probe has its own channels:

```bash
curl -X PUT http://localhost:3001/api/admin/synthetic-monitoring/probes/$PROBE_ID/alert-channels \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": ["ops@example.com"]}'
```

The body is the channels object itself, or `null` to clear it. The answer,
like `GET` on the same path, carries `channels` and `licensed`. When the
probe raises an alert, the channels receive
`Probe "Public site" is down: 3 failed checks in a row.`, and the first
passing run after that resolves the alert and sends
`Probe "Public site" recovered.`

## Rules every channel follows

- Every URL must be an absolute `https://` URL of at most 2,048 characters.
  One that points at a private, loopback or link-local address, or at a name
  that resolves to one, answers `422` when you save it, and every delivery
  checks again.
- A PagerDuty routing key is the 32-character integration key of an Events
  API v2 integration.
- The URLs and the routing key are credentials, so they are encrypted at rest
  and masked in every answer, keeping the host and the last four characters.
  Storing one needs `ENCRYPTION_KEY`. Email addresses are stored readable, so
  a privacy erasure request finds them.
- Sending a masked value back keeps the stored one, so a form that resends
  what it read changes nothing.
- A Slack, Discord, PagerDuty or webhook delivery that fails is retried once.
  A channel that still cannot be reached misses that alert, and the other
  channels still receive theirs.

## Verify a webhook alert

The write that sets a new `webhook_url` answers its signing secret once:
`alert_signing_secret` on a job, `webhook_signing_secret` on the other three.
An unchanged URL keeps its secret, and a new URL gets a new one. Each call to
the webhook carries:

| Header | Value |
|---|---|
| `X-Lyeve-Timestamp` | Unix seconds |
| `X-Lyeve-Signature` | `sha256=` and the hex HMAC-SHA256 of the timestamp, a `.` and the raw body, keyed with the signing secret |

Recompute the signature over the raw body before you parse it, compare in
constant time, and refuse a timestamp more than a few minutes old.

| `event` | Other fields |
|---|---|
| `cron.job.failed`, `cron.job.recovered` | `job_id`, `job_name`, `tenant_id`, `status_code`, `error`, `consecutive_failures`, `time` |
| `error_tracking.spike` | `tenant_id`, `count`, `baseline_mean`, `baseline_stddev`, `z_score`, `window_start`, `window_end`, `top_fingerprint`, `top_fingerprint_count`, `sample`, `detected_at` |
| `logging.volume.exceeded` | `rule_id`, `level`, `tenant_id`, `plugin`, `window`, `count`, `max_count`, `exceeded_by`, `time` |
| `synthetic.probe.down`, `synthetic.probe.recovered` | `probe_id`, `probe_name`, `probe_type`, `tenant_id`, `consecutive_failures`, `error`, `time` |

A field with no value is left out.

## PagerDuty incidents

A failing job and a probe that goes down each open one incident, and their
recovery resolves it. An error spike opens one incident per tenant and a log
volume rule one per rule, so repeated alerts fold into the open incident.
Resolve those in PagerDuty.

## A license from before Alerts Pro

Before these alerts covered errors, logs and probes, they were sold for
scheduled jobs alone as Cron Failure Alerts, and a license bought then carries
the name `cron-pro`. The engine accepts `cron-pro` as `alerts-pro` through the
next minor release, so that license opens every channel on this page. A
license issued now carries `alerts-pro` in its place.

## If the license lapses

Channels set while the license was active keep sending, so you keep hearing
about problems. Changing the email list, removing a channel, clearing a
threshold and sending back the stored setting stay free. Adding or changing a
Slack, Discord, PagerDuty or webhook channel, or setting a spike threshold,
needs `alerts-pro` again. Error event reads go back to the last 30 days, and
the lapse deletes nothing. A super admin can also withhold `alerts-pro` from
one tenant, as [Tenants](tenants.md#choose-what-each-tenant-may-use)
describes, and that tenant then meets the same `402`.

## Errors

| Status | Message | Cause |
|---|---|---|
| `402` | `payment_required`, naming `feature:alerts-pro` | The request adds or changes a paid channel or a spike threshold without `alerts-pro`. Nothing is stored. |
| `422` | `invalid alerts: failure_threshold must be between 1 and 100` | A job's threshold is out of range. |
| `422` | `invalid alert channels: at most 10 email recipients` | Too many addresses. |
| `422` | `invalid alert channels: "<value>" is not an email address` | An address does not parse. |
| `422` | `invalid alert channels: slack_url must be an absolute https:// URL` | A URL is not `https://`. The message names the field. |
| `422` | `invalid alert channels: webhook_url must point to a public address this instance sends to` | The URL points at a private address. |
| `422` | `invalid alert channels: pagerduty_routing_key must be the 32 character key of an Events API v2 integration` | The routing key has the wrong shape. |
| `422` | `... is a masked value that is not stored here, so send the whole value` | A masked value that does not match the stored one. |
| `422` | `spike_min_count must be null or a whole number from 1 to 1000000`, `spike_z_score must be null or a number from 0.5 to 20` | A spike threshold is out of range. |
| `503` | `alert channels other than email cannot be stored right now` | The instance has no `ENCRYPTION_KEY`. Email still saves. |

A body that breaks a rule answers `422` before the license is read, so fix the
fields first. The 402 body in full:

```json
{"error": "payment_required", "plugin": "cron", "feature": "feature:alerts-pro", "upgrade_url": ""}
```

`plugin` names the feature that answered: `cron`, `error-tracking`, `logging`
or `synthetic-monitoring`.

## Related

- [Scheduled jobs](scheduled-jobs.md): schedules, runs, history and the
  `cron.job.failed` event.
- [Error tracking](error-tracking.md): grouping, triage and the spike check.
- [Logs](logs.md): volume rules and the live tail.
- [Synthetic monitoring](synthetic-monitoring.md): probes and their alerts.
- [Email](email.md): the transport the email channel sends through.
