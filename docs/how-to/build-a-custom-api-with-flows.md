---
title: Build a custom API with flows
description: "Join three content types into one GET endpoint with a flow, then test it, cache it, rate limit it, call it with an API key and copy it."
sidebar:
  order: 10
---

> **Included free** on every install. A custom path, other protocols and the outbound nodes require a license with the `flow-pro` feature. See [pricing](https://lyeve.com/pricing/).

Your frontend wants every open order with its customer and its shipments in
one call. This guide builds that endpoint as a [flow](../features/flows.md):
`GET /api/v1/flows/orders-with-customer`, reading three content types. You test
it against real entries before it goes live, cache the answer for 30 seconds,
allow each caller 120 calls a minute, call it with an API key, and export it as
a file you can import anywhere. Everything here runs on the free tier, and
every step works in the admin console or with the API.

## Before you start

- A running instance. The [quickstart](../getting-started/quickstart.md) starts
  one and creates the first administrator.
- For the console steps, the admin console signed in as that administrator.
- For the API steps, `curl`, `jq` and an admin token in `TOKEN`.
  [Sign in](../getting-started/quickstart.md#3-sign-in) shows how to get one.
- Admin calls go to the Admin API on port `3001`, and the finished endpoint is
  served by the Content API on port `3002`. See [the two APIs](../concepts/index.md).

## 1. Create the three content types

An order belongs to a customer, and a shipment belongs to an order.

<div class="paths">
<div data-path="console">

1. Open **Schema builder** and choose **New schema**. Enter `Customer` in
   **Display name** and `customers` in **Machine name**.
2. Choose **Add field**, name it `name`, keep the type `text` and tick **Req**.
   Choose **Add field** again, name it `email` and pick the type `email`.
   Choose **Save**.
3. Choose **New schema** for `Order`, machine name `orders`. Choose
   **Add relation** and name the field `customer`. Under **Relation**, keep
   **Belongs to one**, pick **Customer** in **Target schema** and tick **Req**.
4. Add a `text` field named `state` and tick **Idx**, then a `number` field
   named `total`. Choose **Save**.
5. Choose **New schema** for `Shipment`, machine name `shipments`. Add a
   relation named `order` that belongs to one **Order**, with **Req** ticked,
   then two `text` fields, `carrier` and `tracking_number`. Choose **Save**.

The schema builder in console v0.21.0 has no switch for a unique field, so
`email` is not unique on this path. Nothing below depends on it.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "customers", "display_name": "Customer", "fields": [
        {"name": "name", "field_type": "text", "required": true},
        {"name": "email", "field_type": "email", "unique": true}]}'

curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "orders", "display_name": "Order", "fields": [
        {"name": "customer", "field_type": "relation", "relation_to": "customers", "relation_type": "belongs_to", "required": true},
        {"name": "state", "field_type": "text", "indexed": true},
        {"name": "total", "field_type": "number"}]}'

curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "shipments", "display_name": "Shipment", "fields": [
        {"name": "order", "field_type": "relation", "relation_to": "orders", "relation_type": "belongs_to", "required": true},
        {"name": "carrier", "field_type": "text"},
        {"name": "tracking_number", "field_type": "text"}]}'
```

Each call answers `200` with the content type as stored.

</div>
</div>

The `customer` relation gives every order a `customer_id` field, and `order`
gives every shipment an `order_id` field. The flow reads those two fields.
[Data model](../concepts/data-model.md#relations) explains relations.

## 2. Add a customer, an order and a shipment

The test run in step 5 needs entries to return.

<div class="paths">
<div data-path="console">

1. Open **Content**, pick **Customer** and choose **New entry**. Enter
   `Ada Lovelace` in **Name** and `ada@example.com` in **Email**, then choose
   **Create**.
2. Pick **Order** and choose **New entry**. In **Customer**, type `Ada` and
   pick the customer. Enter `open` in **State** and `42` in **Total**, then
   choose **Create**.
3. Pick **Shipment** and choose **New entry**. Choose **Order** and pick the
   order. An order has no title field, so the picker lists it as
   `Untitled entry`. Enter `DHL` in **Carrier** and `JD0001` in
   **Tracking number**, then choose **Create**.

</div>
<div data-path="api">

```bash
CUSTOMER_ID=$(curl -s -X POST http://localhost:3002/api/v1/content/customers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"name": "Ada Lovelace", "email": "ada@example.com"}}' | jq -r .id)

ORDER_ID=$(curl -s -X POST http://localhost:3002/api/v1/content/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"customer_id": "'"$CUSTOMER_ID"'", "state": "open", "total": 42}}' | jq -r .id)

curl -X POST http://localhost:3002/api/v1/content/shipments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"order_id": "'"$ORDER_ID"'", "carrier": "DHL", "tracking_number": "JD0001"}}'
```

The last call answers `201` with the shipment, its `order_id` set to the order.

</div>
</div>

## 3. Create the account that publishes the flow

The flow's content nodes read as the account that publishes it, and a test
run reads as the account that runs it. The first administrator belongs to no
tenant, so a flow it publishes stops at its first content node with
`the flow's publisher is not an active account in this tenant`. Create an
admin of the tenant for the flow, and let the `admin` role read the three
content types. A free install holds three accounts with an admin role, the
first administrator included.

<div class="paths">
<div data-path="console">

1. Open **Access > Users** and choose **New user**. Enter
   `flows@example.com` in **Email** and a password in **Password**. Under
   **Roles**, untick **editor** and tick **admin**, then choose **Create**.
2. Open **Access > Permissions** and choose **Add a schema rule for admin**.
   Pick `customers` in **Resource**, tick **Read** and choose **Create**.
   Do the same for `orders` and `shipments`.
3. Sign out, and sign in again as `flows@example.com`.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "flows@example.com", "password": "Flows-Publisher-2026", "roles": ["admin"]}'

for type in customers orders shipments; do
  curl -X POST http://localhost:3001/api/admin/permissions \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"role": "admin", "schema_name": "'"$type"'", "actions": ["read"]}'
done
```

The user call answers `201` and each rule `200`. Sign in as the new account
and keep its token in `TOKEN` for the rest of the guide:

```bash
curl -X POST http://localhost:3001/api/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "flows@example.com", "password": "Flows-Publisher-2026"}'

export TOKEN="<the token from the answer>"
```

</div>
</div>

[Access rules](../features/access-rules.md) explains how rules combine.

## 4. Create the flow

The flow has an HTTP trigger and four nodes:

| Node | Type | What it does |
|---|---|---|
| `orders` | `content.query` | Lists orders whose `state` is `?state=` from the URL, or `open`. `populate` embeds each order's customer. |
| `shipments` | `content.query` | Lists the shipments of every loaded order in one query. A list in `filters` matches any of its values, and `map(input, .id)` is the list of order ids. |
| `join` | `data.join` | Attaches each order's shipments to it as `shipments`. `how: left` keeps orders that have none yet. |
| `respond` | `response` | Answers the caller with the joined list. |

The edge from `orders` to `shipments` is what lets `shipments` read the orders
as its `input`: a node can read only what reaches it through an edge.

Save the definition as `orders-with-customer.json`:

```json
{
  "version": 1,
  "name": "Orders with customer and shipments",
  "slug": "orders-with-customer",
  "trigger": { "type": "trigger.http", "config": { "method": "GET", "auth": "auth" } },
  "nodes": [
    { "id": "orders", "type": "content.query", "position": { "x": 120, "y": 200 },
      "config": { "schema": "orders", "filters": { "state": "{{ trigger.query.state ?? \"open\" }}" },
                  "populate": ["customer"], "limit": 100, "sort": "-created_at" } },
    { "id": "shipments", "type": "content.query", "position": { "x": 420, "y": 420 },
      "config": { "schema": "shipments", "filters": { "order_id": "{{ map(input, .id) }}" } } },
    { "id": "join", "type": "data.join", "position": { "x": 420, "y": 300 },
      "config": { "left_key": "id", "right_key": "order_id", "how": "left", "as": "shipments", "many": true } },
    { "id": "respond", "type": "response", "position": { "x": 720, "y": 300 },
      "config": { "status": 200, "body": "{{ input }}" } }
  ],
  "edges": [
    { "from": "orders", "to": "shipments" },
    { "from": "orders", "to": "join", "to_port": "left" },
    { "from": "shipments", "to": "join", "to_port": "right" },
    { "from": "join", "to": "respond" }
  ]
}
```

Then import it as a draft.

<div class="paths">
<div data-path="console">

Open **Flows** and choose **Import**. Upload `orders-with-customer.json` under
**Definition file**, keep **Create a new flow** and choose **Import**. The
editor opens on the draft, with the four nodes drawn on the canvas.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/flows/import \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @orders-with-customer.json
```

The answer is `201` with the flow, `"status": "draft"`, `"version": 0` and
`"unresolved_datasources": []`. Copy its `id` into `FLOW_ID`. The flow now
appears under **Flows** in the admin console, drawn on the canvas.

</div>
</div>

## 5. Test it

A test run executes the draft against a trigger payload you supply. Nothing is
published.

<div class="paths">
<div data-path="console">

On the **Test** tab below the canvas, add `"query": {"state": "open"}` to the
JSON in **Trigger** and choose **Run test**. The run reads `succeeded`, with
the steps `trigger`, `orders`, `shipments`, `join` and `respond` listed beside
it. Choose a step to see what went in and what came out.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/flows/$FLOW_ID/test \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"trigger": {"query": {"state": "open"}}}'
```

The answer is the run, with the response body under `output` and one entry per
step. Trimmed, it looks like this:

```json
{
  "status": "succeeded",
  "is_test": true,
  "output": [
    {
      "id": "abab67c8-e941-4fb3-8d43-25bd70e46025",
      "state": "open",
      "total": 42,
      "customer_id": "e3b09dae-3674-4cf6-8a55-2f4a2b7df61c",
      "customer": { "id": "e3b09dae-3674-4cf6-8a55-2f4a2b7df61c", "name": "Ada Lovelace", "email": "ada@example.com" },
      "shipments": [ { "id": "2f460c41-14b5-49f4-a433-7b2dd0f9162f", "order_id": "abab67c8-e941-4fb3-8d43-25bd70e46025", "carrier": "DHL", "tracking_number": "JD0001" } ]
    }
  ],
  "steps": [
    { "node_id": "trigger", "node_type": "trigger.http", "status": "succeeded" },
    { "node_id": "orders", "node_type": "content.query", "status": "succeeded" },
    { "node_id": "shipments", "node_type": "content.query", "status": "succeeded" },
    { "node_id": "join", "node_type": "data.join", "status": "succeeded" },
    { "node_id": "respond", "node_type": "response", "status": "succeeded" }
  ]
}
```

Each step also carries its `input`, `output`, `error` and `duration_ms`.

</div>
</div>

## 6. Add a cache and a rate limit

Both live on the trigger.

<div class="paths">
<div data-path="console">

Choose the **API request** node. In the **Inspector**, under **Cache**, enter
`30s` in **ttl**. Under **Rate limit**, enter `120` in **requests** and `1m`
in **per**. Choose **Save**.

</div>
<div data-path="api">

This reads the draft, adds them, and saves it back:

```bash
curl -s http://localhost:3001/api/admin/flows/$FLOW_ID \
  -H "Authorization: Bearer $TOKEN" \
  | jq '{definition: (.draft | .trigger.config += {
          cache: {ttl: "30s"},
          rate_limit: {requests: 120, per: "1m"}})}' \
  | curl -X PUT http://localhost:3001/api/admin/flows/$FLOW_ID \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d @-
```

The answer is `200` with the saved draft.

</div>
</div>

The cache keeps each answer for 30 seconds, one entry per URL and caller, so
`?state=open` and `?state=paid` never share one. The rate limit gives each
caller 120 calls a minute. To key either one on something else, add a `key`
expression, as
[Limit and cache an endpoint](../features/flows.md#limit-and-cache-an-endpoint)
explains.

## 7. Publish

<div class="paths">
<div data-path="console">

Choose **Publish**. The badge beside the slug reads **Published v1**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/flows/$FLOW_ID/publish \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `200` with `"status": "active"` and `"version": 1`.

</div>
</div>

Later saves change only the draft, so the endpoint keeps serving version 1
until you publish again.

## 8. Create an API key for the frontend

A key that calls flows needs the `flows:read` scope.

<div class="paths">
<div data-path="console">

The **API keys** page is open to a super admin, so sign in again as the first
administrator.

1. Open **Access > API keys** and choose **New key**.
2. Enter `orders-frontend` in **Name**.
3. Under **Content**, untick **Read**. Open **Schema catalog** and untick
   **Discover content types**. Open **Flows** and tick **Read**. The summary
   below the sections reads `1 scope: flows:read`.
4. Choose **Create**.

The dialog shows the key once. Copy it into `API_KEY` for the next step.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/api-keys \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "orders-frontend", "scopes": ["flows:read"]}'
```

The answer is `201` and carries the key once, as `raw_key`. Copy it into
`API_KEY`.

</div>
</div>

See [API keys](../features/api-keys.md) for scopes and expiry.

## 9. Call your endpoint

This step is the same on both paths:

```bash
curl -i "http://localhost:3002/api/v1/flows/orders-with-customer?state=open" \
  -H "X-API-Key: $API_KEY"
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Ratelimit-Limit: 120
Ratelimit-Remaining: 119
Ratelimit-Reset: 1790917634
X-Flow-Cache: miss
```

The body is the same list the test returned. Send the call again within 30
seconds and `X-Flow-Cache` says `hit`: the flow did not run. Past 120 calls in
a minute, a call answers `429` with a `Retry-After` header. A call with no credentials
answers `401`.

## 10. Export it and import a copy

Download the flow as YAML, then import it as a new draft under another slug.

<div class="paths">
<div data-path="console">

1. Open the flow, choose **More actions** (the `...` button beside **Flows**)
   and then **Export**. Pick **YAML** and choose **Download YAML**.
2. In the downloaded `orders-with-customer.yaml`, change the line
   `slug: orders-with-customer` to `slug: orders-v2`. The import dialog has no
   slug field of its own.
3. Open **Flows**, choose **Import**, upload the file and choose **Import**.

The editor opens on `orders-v2` as a draft.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/flows/$FLOW_ID/export?format=yaml" \
  -H "Authorization: Bearer $TOKEN" -o orders-with-customer.yaml

curl -X POST "http://localhost:3001/api/admin/flows/import?mode=create&slug=orders-v2" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@orders-with-customer.yaml"
```

The import answers `201` with the new flow as a `draft`, and
`"unresolved_datasources": []`, because this flow uses no saved connection.

</div>
</div>

The same file imports into another instance, which is how you promote a flow
from staging to production. To import into another tenant of this instance,
send the import as a super admin with `X-Tenant-ID` naming that tenant, as
[Tenants](../features/tenants.md#how-a-request-picks-its-tenant) explains. See
[Move flows between tenants and instances](../features/flows.md#move-flows-between-tenants-and-instances).

## Next

- [Flow templates](flow-templates.md): eighteen starter flows, among them a
  larger version of this one. It filters on a `status` field, so change its
  `orders` filter to `state` to run it on the types from this guide.
- [Keep the content model and flows in git](schema-and-flows-as-code.md):
  review flow files in pull requests and promote them on release.
- [Flows](../features/flows.md#your-own-endpoint-path): serve the endpoint at a
  path of your own, or over GraphQL, gRPC and realtime, with `flow-pro`.
- [Flow definition format](../reference/flow-definition.md): every node, its
  settings and the expression language.
