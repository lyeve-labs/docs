---
title: "12. Webhooks, events and email"
description: "Send trip changes to another service with a webhook, take in payloads from outside, replay the event log, and send mail through a provider, a template and a trigger."
sidebar:
  order: 12
covers:
  console:
    - "/admin/webhooks"
    - "/admin/webhooks/incoming"
    - "/admin/events"
    - "/admin/settings/email"
    - "/admin/settings/email/templates"
    - "/admin/settings/email/triggers"
  api:
    - "POST /api/admin/webhooks"
    - "POST /api/admin/webhooks/{id}/test"
    - "GET /api/admin/webhooks/{id}/deliveries/stats"
    - "POST /api/admin/incoming-webhooks"
    - "POST /api/v1/webhooks/in/{id}"
    - "GET /api/admin/events"
    - "POST /api/admin/events/replay"
    - "POST /api/admin/email/providers"
    - "POST /api/admin/email-templates"
    - "POST /api/admin/email/triggers"
verified:
  engine: v0.62.0
  console: v0.28.0
---

A change to a trip can reach other services on its own. In this lesson you send each trip
change to an address you control, take a payload from another service in as a new trip,
replay what was published, and send mail when a trip changes. You need the `trip` content
type from lesson 3 and the token from [lesson 11](graphql-and-streams.md). Creating and
changing a webhook takes the `super_admin` role.

## Send trip changes to another service

> **Included free** on every install, up to 25 webhooks per tenant.

Use an HTTPS address you control. Its host must resolve to a public address when you save.

<div class="paths">
<div data-path="console">

1. Open **Delivery > Webhooks** and choose **New webhook**.
2. Enter `Trip changes` in **Name** and your address in **URL**. Enter a long random value
   in **Signing secret**, which signs each payload with HMAC-SHA256.
3. Under **Events**, tick `after_create`, `after_update` and `after_delete`.
4. In **Content types**, which reads **Every content type**, pick `trip`.
5. Keep **Enabled** on and choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Trip changes",
        "url": "https://hooks.example.com/lyeve",
        "events": ["after_create", "after_update", "after_delete"],
        "schemas": ["trip"],
        "secret": "replace-with-a-long-random-secret"
      }'
```

The answer is `201` with the webhook and its `id`, which never shows the secret. Copy the
`id` into `WEBHOOK_ID`.

</div>
</div>

![Opening New webhook, filling in the name, URL and secret, ticking three events, picking trip and creating it.](./media/webhooks-and-email-new-webhook.mp4)

The form has more than the basics:

| Part | What it does |
|---|---|
| **Events** | Which writes send a delivery |
| **Content types** | Which content types it watches. Empty means every one |
| **Delivery options** | **Payload template**, **JSONPath filter**, **Field filters**, **Retries** and **Retry delay (seconds)**. Empty uses the default |
| **Enabled** | Off keeps the webhook and sends nothing |

The delivery options need the `webhook-pro` feature. On a free install the form still shows
them, and an option that is not enabled on this engine cannot be saved.

## Test it and read the history

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. On the webhook's row, choose **Test webhook**, the flask icon. The outcome shows under
   the row, such as `HTTP 204: No Content`.
2. Choose **Delivery history**, the clock icon, to list each delivery with its status and
   result.

**Rotate signing secret**, the key icon, issues a new secret and shows it once. The old one
stops working at once, so copy the new one to the receiver.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/webhooks/$WEBHOOK_ID/test \
  -H "Authorization: Bearer $TOKEN"

curl http://localhost:3001/api/admin/webhooks/$WEBHOOK_ID/deliveries/stats \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![The webhook row with a test result under it, and the delivery history open.](./media/webhooks-and-email-test.png)

[Add a webhook](../how-to/add-a-webhook.md) writes the receiver that checks the signature.
**Dead letters**, at the top of the page, lists the deliveries that failed every retry.

## Take in a payload from another service

> **Requires** a license with the `webhook-pro` feature.

An incoming webhook turns a JSON payload from another service into a new entry.

<div class="paths">
<div data-path="console">

1. Open **Delivery > Incoming webhooks** and choose **New incoming webhook**.
2. Enter `Partner trips` in **Name** and pick `trip` in **Target content type**.
3. Enter a long random secret in **Signing secret**, and give it to the other service.
4. In **Field map**, enter `{"name": "title"}`, which maps the payload's `name` to the
   trip's `title`.
5. Leave **IP allowlist** empty to take requests from any address, then choose **Create**.

The list shows the address the other service posts to, `POST /api/v1/webhooks/in/` and the
webhook's id.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/incoming-webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Partner trips",
        "secret": "replace-with-a-long-random-secret",
        "schema_name": "trip",
        "field_map": {"name": "title"}
      }'
```

The other service then posts to `/api/v1/webhooks/in/{id}` on the Content API, with no
credential, signed with the secret.

</div>
</div>

On a free install the page says that creating and changing incoming webhooks is not enabled
on this engine, and you can still delete the ones you have.

![The Incoming webhooks page on a free install, with the notice and no New incoming webhook button.](./media/webhooks-and-email-incoming-free.png)

![The Incoming webhooks page on a licensed install, with the Partner trips endpoint and its address.](./media/webhooks-and-email-incoming-licensed.png)

[Webhooks](../features/webhooks.md#take-in-webhooks-from-other-services) shows how the other
service signs its request.

## Replay what was published

> **Requires** a license with the `events` feature.

The event log keeps every published event. A replay publishes the events of a time window
again, to webhooks, flows, the message broker, realtime clients and email triggers.

<div class="paths">
<div data-path="console">

1. Open **Insight > Events**. **Published** lists the log, and **Only events with errors**
   narrows it.
2. Under **Replay**, set **Replay from** and enter `trip-backfill` in **Run name**.
3. Keep **Rehearse without publishing** ticked and choose **Rehearse**. The run counts the
   events and publishes nothing.
4. Untick it and choose **Replay**. The console asks first, naming the window and the run.
   Choose **Replay** again to publish for real.

Each run is listed with its status, progress and result. A later run under the same name
skips the events that name already replayed.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/events/replay \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"since": "2026-11-01T00:00:00Z", "handler": "trip-backfill", "dry_run": true}'
```

The answer is `202`. `GET /api/admin/events` lists the log.

</div>
</div>

A real replay cannot be undone, and every listener gets the events again.

![The Events page with a rehearsal and a real run of trip-backfill listed under Replay.](./media/webhooks-and-email-replay.png)

## Add an email provider

> **Included free** on every install, with one SMTP provider per tenant. More providers and
> the Provider API transport need a license with the `email-pro` feature.

<div class="paths">
<div data-path="console">

1. Open **Settings > Email** and choose **New provider**.
2. Enter `primary` in **Name**, keep **SMTP** as the **Transport** and enter
   `no-reply@example.com` in **From address**.
3. Enter your relay's **Host**, `587` in **Port**, and its **Username** and **Password**.
   Leave **Implicit TLS** off, because port 587 starts in plain text.
4. Keep **Priority** at `0` and choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/email/providers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "primary",
        "transport": "smtp",
        "host": "smtp.example.com",
        "port": 587,
        "username": "mailer",
        "password": "use-a-secret-manager",
        "from_addr": "no-reply@example.com",
        "priority": 0
      }'
```

</div>
</div>

![Opening New provider, filling in the SMTP server and creating it.](./media/webhooks-and-email-provider.mp4)

**Fallback transport**, below the pool, holds the engine settings used when the pool is empty
or every provider fails. A change there applies at the next engine restart.

## Write a template

> **Included free** on every install, with five templates per tenant.

<div class="paths">
<div data-path="console">

1. On **Settings > Email**, choose **Templates**, then **New template**.
2. Keep **A blank template** in **Start from**, enter `trip-updated` in **Name** and pick
   **Active** in **Status**.
3. Enter `{{.Vars.title}} has changed` in **Subject**, and an MJML body in **Body** that
   prints `{{.Vars.title}}`.
4. Choose **Create**. The row's preview button renders the template with sample values.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/email-templates \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key": "trip-updated", "status": "active", "subject": "{{.Vars.title}} has changed", "mjml_source": "<mjml><mj-body><mj-section><mj-column><mj-text>{{.Vars.title}} was updated.</mj-text></mj-column></mj-section></mj-body></mjml>"}'
```

</div>
</div>

Two templates are marked required, because password resets and sign-in links use them, and
they cannot be deleted. When every slot is in use, the page says so.

## Mail when a trip changes

> **Requires** a license with the `email-pro` feature to create or edit a trigger. Listing
> and deleting are free.

<div class="paths">
<div data-path="console">

1. On **Settings > Email**, choose **Triggers**, then **New trigger**.
2. Enter `Trip updated` in **Name**, pick **A record is updated** in **When** and `trip` in
   **In**.
3. Pick `trip-updated` in **Template** and enter your address in **Send to**.
4. Keep **Enabled** on and choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/email/triggers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Trip updated", "event": "after_update", "schema": "trip", "template_key": "trip-updated", "to_static": ["you@example.com"], "enabled": true}'
```

</div>
</div>

On a free install the page says that creating or editing a trigger is not enabled on this
engine. A tenant with no active template is told to create one first.

![The Email triggers page on a free install, with the notice in place of New trigger.](./media/webhooks-and-email-triggers-free.png)

![Creating the Trip updated trigger on a licensed install.](./media/webhooks-and-email-triggers-licensed.mp4)

## Next

[13. Flows](flows.md) builds an endpoint that reads trips. For depth, read
[Webhooks](../features/webhooks.md), [Event replay](../features/event-replay.md) and
[Email](../features/email.md).
