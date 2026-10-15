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
a file you can import anywhere. Everything here runs on the free tier.

## Before you start

- A running instance and an admin token in `TOKEN`. The
  [quickstart](../getting-started/quickstart.md) shows how to get one.
- `curl` and `jq`.
- Admin calls go to the Admin API on port `3001`, and the finished endpoint is
  served by the Content API on port `3002`. See [the two APIs](../concepts/index.md).

## 1. Create the three content types

An order belongs to a customer, and a shipment belongs to an order:

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
        {"name": "status", "field_type": "text", "indexed": true},
        {"name": "total", "field_type": "number"}]}'

curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "shipments", "display_name": "Shipment", "fields": [
        {"name": "order", "field_type": "relation", "relation_to": "orders", "relation_type": "belongs_to", "required": true},
        {"name": "carrier", "field_type": "text"},
        {"name": "tracking_number", "field_type": "text"}]}'
```

Each call answers `200` with the content type as stored. The `customer`
relation gives every order a `customer_id` field, and `order` gives every
shipment an `order_id` field. The flow reads those two fields.
[Data model](../concepts/data-model.md#relations) explains relations.

## 2. Add a customer, an order and a shipment

The test run in step 4 needs entries to return:

```bash
CUSTOMER_ID=$(curl -s -X POST http://localhost:3002/api/v1/content/customers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"name": "Ada Lovelace", "email": "ada@example.com"}}' | jq -r .id)

ORDER_ID=$(curl -s -X POST http://localhost:3002/api/v1/content/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"customer_id": "'"$CUSTOMER_ID"'", "status": "open", "total": 42}}' | jq -r .id)

curl -X POST http://localhost:3002/api/v1/content/shipments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"order_id": "'"$ORDER_ID"'", "carrier": "DHL", "tracking_number": "JD0001"}}'
```

The last call answers `201` with the shipment, its `order_id` set to the order.

## 3. Create the flow

The flow has an HTTP trigger and four nodes:

| Node | Type | What it does |
|---|---|---|
| `orders` | `content.query` | Lists orders whose `status` is `?status=` from the URL, or `open`. `populate` embeds each order's customer. |
| `shipments` | `content.query` | Lists the shipments of every loaded order in one query. A list in `filters` matches any of its values, and `map(input, .id)` is the list of order ids. |
| `join` | `data.join` | Attaches each order's shipments to it as `shipments`. `how: left` keeps orders that have none yet. |
| `respond` | `response` | Answers the caller with the joined list. |

The edge from `orders` to `shipments` is what lets `shipments` read the orders
as its `input`: a node can read only what reaches it through an edge.

```bash
curl -X POST http://localhost:3001/api/admin/flows \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Orders with customer and shipments",
    "slug": "orders-with-customer",
    "definition": {
      "version": 1,
      "trigger": { "type": "trigger.http", "config": { "method": "GET", "auth": "auth" } },
      "nodes": [
        { "id": "orders", "type": "content.query", "position": { "x": 120, "y": 200 },
          "config": { "schema": "orders", "filters": { "status": "{{ trigger.query.status ?? \"open\" }}" },
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
  }'
```

The answer is `201` with the flow, `"status": "draft"` and `"version": 0`.
Copy its `id` into `FLOW_ID`. The flow now appears under **Flows** in the admin
console, drawn on the canvas.

## 4. Test it

A test run executes the draft against a trigger payload you supply. Nothing is
published:

```bash
curl -X POST http://localhost:3001/api/admin/flows/$FLOW_ID/test \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"trigger": {"query": {"status": "open"}}}'
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
      "status": "open",
      "total": "42",
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

:::caution
Content nodes return number fields as strings, so `total` reads `"42"` here
while the Content API returns `42`. Convert it before you compare or add
numbers, in your frontend or in the flow with an expression such as
`{{ float(item.total) }}`.
:::

In the editor, the same run is on the **Test** tab: type the trigger payload, press
**Run test**, and click a node to see what went in and what came out.

## 5. Add a cache and a rate limit

Both live on the trigger. This reads the draft, adds them, and saves it back:

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

The answer is `200` with the saved draft. The cache keeps each answer for 30
seconds, one entry per URL and caller, so `?status=open` and `?status=paid`
never share one. The rate limit gives each caller 120 calls a minute. To key
either one on something else, add a `key` expression, as
[Limit and cache an endpoint](../features/flows.md#limit-and-cache-an-endpoint)
explains.

## 6. Publish

```bash
curl -X POST http://localhost:3001/api/admin/flows/$FLOW_ID/publish \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `200` with `"status": "active"` and `"version": 1`. Later saves
change only the draft, so the endpoint keeps serving version 1 until you
publish again.

## 7. Create an API key for the frontend

A key that calls flows needs the `flows:read` scope:

```bash
curl -X POST http://localhost:3001/api/admin/api-keys \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "orders-frontend", "scopes": ["flows:read"]}'
```

The answer is `201` and carries the key once, as `raw_key`. Copy it into
`API_KEY`. See [API keys](../features/api-keys.md) for scopes and expiry.

## 8. Call your endpoint

```bash
curl -i "http://localhost:3002/api/v1/flows/orders-with-customer?status=open" \
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

## 9. Export it and import a copy

Download the flow as YAML, then import it as a new draft under another slug:

```bash
curl "http://localhost:3001/api/admin/flows/$FLOW_ID/export?format=yaml" \
  -H "Authorization: Bearer $TOKEN" -o orders-with-customer.yaml

curl -X POST "http://localhost:3001/api/admin/flows/import?mode=create&slug=orders-v2" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@orders-with-customer.yaml"
```

The import answers `201` with the new flow as a `draft`, and
`"unresolved_datasources": []`, because this flow uses no saved connection.
The same file imports into another instance, which is how you promote a flow
from staging to production. To import into another tenant of this instance,
send the import as a super admin with `X-Tenant-ID` naming that tenant, as
[Tenants](../features/tenants.md#how-a-request-picks-its-tenant) explains. See
[Move flows between tenants and instances](../features/flows.md#move-flows-between-tenants-and-instances).

## Next

- [Flow templates](flow-templates.md): eighteen starter flows, among them a
  larger version of this one.
- [Keep the content model and flows in git](schema-and-flows-as-code.md):
  review flow files in pull requests and promote them on release.
- [Flows](../features/flows.md#your-own-endpoint-path): serve the endpoint at a
  path of your own, or over GraphQL, gRPC and realtime, with `flow-pro`.
- [Flow definition format](../reference/flow-definition.md): every node, its
  settings and the expression language.
