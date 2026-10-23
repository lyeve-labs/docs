---
title: Add a webhook
description: "Run a receiver that checks LyEve's signature, register it, send a test and a real event, read the history and recover missed deliveries."
sidebar:
  order: 6
---

> **Included free** on every install, up to 25 webhooks per tenant. Step 6 sets a retry policy and resends a delivery, which need a license with the `webhook-pro` feature, and step 7 needs `events`. See [pricing](https://lyeve.com/pricing/).

This walkthrough connects LyEve to an endpoint of your own. You run a small
receiver that refuses any request LyEve did not sign, register it as a
webhook, watch a test delivery and a real content event arrive, then read the
history and recover what failed. Every field, header and route is described on
[Webhooks](../features/webhooks.md). This page is the path through them.

## Before you start

- An account with the `super_admin` role, such as the first administrator.
  For the console steps, sign in to the admin console with it. For the API
  steps, keep its token in `TOKEN`.
  [Sign in](../getting-started/quickstart.md#3-sign-in) shows how to get one.
- A content type to watch. The steps use `post` from
  [Create your first content type](../getting-started/first-content-type.md).
- Node.js 18 or later, or Python 3.8 or later. Neither receiver needs a
  package.
- A place the instance can reach your receiver. The URL's host must resolve to
  a public address, or to a private range the operator lists in
  `WEBHOOK_ALLOWED_PRIVATE_NETWORKS`.

## 1. Run a receiver

Pick a long random secret and keep it in `WEBHOOK_SECRET`. LyEve signs every
delivery with it, and the receiver recomputes the signature over the raw body,
the timestamp and the nonce. It also refuses a timestamp older than five
minutes and a nonce it has already seen, so a captured request cannot be
replayed.

Node.js, saved as `receiver.mjs`:

```javascript
import http from "node:http";
import crypto from "node:crypto";

const SECRET = process.env.WEBHOOK_SECRET;
const PORT = Number(process.env.PORT ?? 8787);
const MAX_AGE_SECONDS = 300;
const seenNonces = new Map(); // nonce -> time it may be forgotten

function verify(headers, rawBody) {
  const timestamp = headers["x-webhook-timestamp"];
  const nonce = headers["x-webhook-nonce"];
  const signature = headers["x-webhook-signature"];
  if (!timestamp || !nonce || !signature) return "unsigned request";

  const now = Date.now() / 1000;
  if (Math.abs(now - Number(timestamp)) > MAX_AGE_SECONDS) return "stale timestamp";

  for (const [seen, forgetAt] of seenNonces) if (forgetAt < now) seenNonces.delete(seen);
  if (seenNonces.has(nonce)) return "nonce already used";

  const expected = "sha256=" + crypto
    .createHmac("sha256", SECRET)
    .update(`${timestamp}.${nonce}.`)
    .update(rawBody)
    .digest("hex");
  const a = Buffer.from(signature);
  const b = Buffer.from(expected);
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) return "bad signature";

  seenNonces.set(nonce, now + 2 * MAX_AGE_SECONDS);
  return null;
}

http.createServer((req, res) => {
  const chunks = [];
  req.on("data", (chunk) => chunks.push(chunk));
  req.on("end", () => {
    const rawBody = Buffer.concat(chunks);
    const problem = verify(req.headers, rawBody);
    if (problem) {
      console.log("refused:", problem);
      res.writeHead(401).end(problem);
      return;
    }
    const delivery = JSON.parse(rawBody);
    console.log("accepted:", delivery.event, delivery.schema, JSON.stringify(delivery.data));
    res.writeHead(204).end();
  });
}).listen(PORT, () => console.log(`listening on http://localhost:${PORT}`));
```

Python, saved as `receiver.py`:

```python
import hashlib
import hmac
import json
import os
import time
from http.server import BaseHTTPRequestHandler, HTTPServer

SECRET = os.environ["WEBHOOK_SECRET"].encode()
PORT = int(os.environ.get("PORT", "8787"))
MAX_AGE_SECONDS = 300
seen_nonces = {}  # nonce -> time it may be forgotten


def verify(headers, raw_body):
    timestamp = headers.get("X-Webhook-Timestamp")
    nonce = headers.get("X-Webhook-Nonce")
    signature = headers.get("X-Webhook-Signature")
    if not (timestamp and nonce and signature):
        return "unsigned request"

    now = time.time()
    if abs(now - int(timestamp)) > MAX_AGE_SECONDS:
        return "stale timestamp"

    for seen in [n for n, forget_at in seen_nonces.items() if forget_at < now]:
        del seen_nonces[seen]
    if nonce in seen_nonces:
        return "nonce already used"

    signed = f"{timestamp}.{nonce}.".encode() + raw_body
    expected = "sha256=" + hmac.new(SECRET, signed, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, signature):
        return "bad signature"

    seen_nonces[nonce] = now + 2 * MAX_AGE_SECONDS
    return None


class Receiver(BaseHTTPRequestHandler):
    def do_POST(self):
        raw_body = self.rfile.read(int(self.headers.get("Content-Length", 0)))
        problem = verify(self.headers, raw_body)
        if problem:
            print("refused:", problem, flush=True)
            self.send_response(401)
            self.end_headers()
            self.wfile.write(problem.encode())
            return
        delivery = json.loads(raw_body)
        data = json.dumps(delivery["data"], separators=(",", ":"), ensure_ascii=False)
        print("accepted:", delivery["event"], delivery["schema"], data, flush=True)
        self.send_response(204)
        self.end_headers()

    def log_message(self, *args):
        pass


print(f"listening on http://localhost:{PORT}", flush=True)
HTTPServer(("", PORT), Receiver).serve_forever()
```

Start one of them:

```bash
export WEBHOOK_SECRET='replace-with-a-long-random-secret'
node receiver.mjs        # or: python3 receiver.py
```

```text
listening on http://localhost:8787
```

Both print the fields of the standard body. If you give the webhook a
`payload_template`, the body is your own JSON, so change the line that prints
it. Both check the raw bytes. If you move the check into a framework, read the body
before any JSON parser touches it (in Express, `express.raw({ type: "application/json" })`),
because a re-serialized body no longer matches the signature. Keep the nonce
list in a shared store such as Redis when you run more than one receiver.

## 2. Register the receiver

Use the address the instance reaches your receiver at, and the same secret.

<div class="paths">
<div data-path="console">

1. Open **Delivery > Webhooks** and choose **New webhook**.
2. Enter `My receiver` in **Name** and the receiver's address in **URL**.
   Enter the value of `WEBHOOK_SECRET` in **Signing secret**.
3. Under **Events**, tick `after_create`, `after_update` and `after_delete`.
4. In **Schemas**, which reads **Every schema**, pick `post`.
5. Choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "My receiver",
        "url": "https://hooks.example.com/lyeve",
        "events": ["after_create", "after_update", "after_delete"],
        "schemas": ["post"],
        "secret": "'"$WEBHOOK_SECRET"'"
      }'
```

The answer is `201` with the webhook. Copy its `id` into `WEBHOOK_ID`. A URL
whose host does not resolve, or resolves to a private address, answers
`400 invalid input`.

</div>
</div>

## 3. Send a test

<div class="paths">
<div data-path="console">

Choose **Test**, the flask icon, on the webhook's row. The outcome shows under
the row: `HTTP 204: No Content`.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/webhooks/$WEBHOOK_ID/test \
  -H "Authorization: Bearer $TOKEN"
```

The API answers `{"success": true, "status_code": 204, "message": "No Content"}`.

</div>
</div>

The receiver prints:

```text
accepted: test * {"message":"webhook test from LyEve"}
```

If the receiver prints `refused: bad signature`, the two secrets differ.

## 4. Fire a real event

<div class="paths">
<div data-path="console">

Open **Content**, pick **Post** and choose **New entry**. Enter
`Webhook check` in **Title** and `Hello` in **Body**, then choose **Create**.

Within a moment the receiver prints an `after_create` line. An entry saved in
console v0.21.0 arrives as the console stores it: the fields you typed sit
under `body`, beside the entry's `id`, `slug`, `status`, `title` and its
timestamps.

</div>
<div data-path="api">

Create a `post` on the Content API:

```bash
curl -X POST http://localhost:3002/api/v1/content/post \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"title": "Webhook check", "slug": "webhook-check", "body": "Hello"}}'
```

Within a moment the receiver prints an `after_create` line with the entry's
fields in alphabetical order, its `id` and status among them:

```text
accepted: after_create post {"_status":"published","body":"Hello","id":"4af50561-8093-4925-983f-ed8813ab615b","slug":"webhook-check","title":"Webhook check"}
```

</div>
</div>

## 5. Read the history

<div class="paths">
<div data-path="console">

Choose **Delivery history**, the clock icon, on the webhook's row. The dialog
lists the test and the `after_create` delivery, newest first, each with its
time, event, content type, status, duration and result.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/webhooks/$WEBHOOK_ID/deliveries?limit=5" \
  -H "Authorization: Bearer $TOKEN"
```

The answer lists the latest deliveries, newest first:

```json
[
  {
    "id": "c1b593c6-8e40-4a97-9851-d545eae5c27e",
    "webhook_id": "df373000-a2df-4fcb-b7c1-949494a57337",
    "tenant_id": "default",
    "event_type": "after_create",
    "schema_name": "post",
    "status_code": 204,
    "success": true,
    "duration_ms": 32,
    "request_body": "{\"event\":\"after_create\",\"schema\":\"post\",\"data\":{\"_status\":\"published\",\"body\":\"Hello\",\"id\":\"4af50561-8093-4925-983f-ed8813ab615b\",\"slug\":\"webhook-check\",\"title\":\"Webhook check\"},\"timestamp\":\"2026-10-03T14:29:09Z\"}",
    "original_payload": "{\"data\": {\"id\": \"4af50561-8093-4925-983f-ed8813ab615b\", \"body\": \"Hello\", \"slug\": \"webhook-check\", \"title\": \"Webhook check\", \"_status\": \"published\"}, \"event\": \"after_create\", \"schema\": \"post\", \"timestamp\": \"2026-10-03T14:29:09Z\"}",
    "attempted_at": "2026-10-03T14:29:09.529085Z",
    "retry_count": 0
  }
]
```

A failed delivery carries `success: false`, the `status_code` or an `error`,
and `next_retry_at`.

</div>
</div>

## 6. Recover a failed delivery

Without a retry policy, the default retries a failed delivery after 30, 60,
120 and 240 seconds, as [Retry policy](../features/webhooks.md#retry-policy)
explains. For shorter waits while you test, set a policy of your own. The
policy, the manual retry and the replay below need `webhook-pro`:

```bash
curl -X PUT http://localhost:3001/api/admin/webhooks/$WEBHOOK_ID/retry-config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"max_attempts": 5, "base_delay_ms": 5000, "max_delay_ms": 20000, "strategy": "exponential"}'
```

Stop the receiver and create another `post`. The delivery fails. With this
policy its four retries wait 5, 10, 20 and 20 seconds, each counted from the
attempt before it. A retry goes at the first 30-second check after its wait
ends, and every wait here is shorter than that, so the retries arrive about
half a minute apart. Start the receiver again and either wait for the next retry,
which carries `X-Webhook-Retry: true`, or send the delivery yourself with its
`id` from the history:

```bash
curl -X POST http://localhost:3001/api/admin/webhooks/$WEBHOOK_ID/deliveries/$DELIVERY_ID/retry \
  -H "Authorization: Bearer $TOKEN"
```

A delivery that ran out of attempts waits in the dead-letter queue. List it
with `GET /api/admin/webhook-dead-letters?status=pending` and send it with
`POST /api/admin/webhook-dead-letters/{id}/replay`, as
[Replay or dismiss a dead letter](../features/webhooks.md#replay-or-dismiss-a-dead-letter)
shows.

## 7. Recover a missed window

When the receiver was down for longer than its retries lasted, send every
content event of the window again with [event replay](../features/event-replay.md).
This step needs a license with the `events` feature. Start with a dry run that
counts what would be sent:

```bash
curl -i -X POST http://localhost:3001/api/admin/events/replay \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"since": "2026-10-01T09:00:00Z", "handler": "my-receiver-catchup", "dry_run": true}'
```

The answer is `202` with a `Location` header naming the run. Read the run until
`status` is `completed`, and check `total_events`:

```bash
curl http://localhost:3001/api/admin/events/replay/$RUN_ID \
  -H "Authorization: Bearer $TOKEN"
```

Send the same request without `dry_run` to replay for real. Each event is
published again, the webhook matches it as it matched the original, and the
receiver gets it with a fresh timestamp and nonce. Running the same `handler`
name again skips events it already replayed.

:::caution
A replay publishes the events to everything that listens, not only this
webhook: other webhooks, flows, a message broker and realtime clients see them
too. The receiver may get an event it already processed, so deduplicate on the
entry's `id` and the event.
:::

## Next

- [Webhooks](../features/webhooks.md): filters, payload templates, health and
  incoming webhooks.
- [Event replay](../features/event-replay.md): the event log and how replays
  run.
- [Flows](../features/flows.md): send your own events with `webhook.deliver`.
