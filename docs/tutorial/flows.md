---
title: "13. Flows"
description: "Build an endpoint that lists trips on the flow canvas, test the draft, publish it and call it, then store values and connections your flows use."
sidebar:
  order: 13
covers:
  console:
    - "/admin/flows"
    - "/admin/flows/[id]"
    - "/admin/flows/datasources"
    - "/admin/flows/variables"
  api:
    - "POST /api/admin/flows"
    - "POST /api/admin/flows/{id}/test"
    - "POST /api/admin/flows/{id}/publish"
    - "GET /api/v1/flows/{slug}"
    - "PUT /api/admin/flows/variables"
    - "POST /api/admin/flows/datasources"
verified:
  engine: v0.62.0
  console: v0.28.0
---

A flow is an endpoint or an automation you draw on a canvas. In this lesson you build
`latest_trips`, an endpoint that answers with the newest trips, test it, publish it and call
it. You need the `trip` content type from lesson 3, a few trips, and the token from
[lesson 12](webhooks-and-email.md).

## Create the flow

> **Included free** on every install, up to twenty flows per tenant.

<div class="paths">
<div data-path="console">

1. Open **Flows** and choose **New flow**.
2. Enter `Latest trips` in **Name** and `latest_trips` in **Slug**. The slug takes lower
   case letters, digits and underscores, and it becomes part of the URL.
3. Choose **Create**.

The editor opens on a draft that holds an **API request** trigger. It answers `GET` calls
from a signed-in user or an API key.

</div>
<div data-path="api">

This creates the whole draft at once, with a node that lists trips and a node that answers:

```bash
curl -X POST http://localhost:3001/api/admin/flows \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Latest trips",
        "slug": "latest_trips",
        "definition": {
          "version": 1,
          "name": "Latest trips",
          "slug": "latest_trips",
          "trigger": { "type": "trigger.http", "config": { "method": "GET", "auth": "auth" } },
          "nodes": [
            { "id": "trips", "type": "content.query",
              "config": { "schema": "trip", "limit": 5, "sort": "-created_at" } },
            { "id": "respond", "type": "response",
              "config": { "status": 200, "body": "{{ input }}" } }
          ],
          "edges": [ { "from": "trips", "to": "respond" } ]
        }
      }'
```

The answer is `201` with the flow. Copy its `id` into `FLOW_ID`.

</div>
</div>

## Draw the nodes

This is a console task. Over the API, the definition you sent above holds the same nodes.

1. In the **Palette**, find **Query content** under **Data** and drag it onto the canvas.
2. Join the trigger to it. In the **Inspector**, pick `trip` as its schema and set the limit
   to `5`.
3. Drag **Response** from **Output** onto the canvas and join **Query content** to it. Set
   its body to `{{ input }}`, which answers with the list the node before it read.
4. Choose **Save**. The badge beside the flow name reads **Saved**.

![Dragging Query content and Response from the palette, joining them to the trigger, and setting the schema to trip in the Inspector.](./media/flows-draw.webm)

The editor has these parts:

| Part | What it does |
|---|---|
| **Palette** | Every node type, by category. A paid node is marked on a free install |
| Canvas | The trigger on the left starts every run. Drag to join nodes |
| **Inspector** | The settings of the selected node |
| **Test** | Runs the draft on a trigger payload you edit |
| **Save** and **Publish** | Keep the draft, or make it the version callers get |
| **Runs** | Every run with its status, trigger, kind and duration |
| **More actions** | Disable the flow, which stops serving the published version, or delete it |

When the canvas is empty, it offers **Start from the palette**, **Use a template** and
**Import YAML or JSON**. A problem in the definition shows in **Fix these before
publishing**, and choosing one jumps to its node.

## Test the draft

<div class="paths">
<div data-path="console">

1. Choose **Test**. **Trigger** holds the payload the run starts with.
2. Keep **Allow side effects** off, so writes and outgoing calls run as dry runs.
3. Choose **Run test**.

The run lists each node with its input, output and timing. The response step holds your
trips.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/flows/$FLOW_ID/test \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"trigger": {}}'
```

The answer's `output` is the list of trips, and `steps` lists every node.

</div>
</div>

![Opening Test, running the draft and reading the trips in the response step.](./media/flows-test.webm)

## Publish it and call it

<div class="paths">
<div data-path="console">

Choose **Publish**. The badge beside the name reads **Published** with the version, and the
flow's row under **Flows** shows the same.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/flows/$FLOW_ID/publish \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

Any client calls the published flow on the Content API:

```bash
curl http://localhost:3002/api/v1/flows/latest_trips \
  -H "Authorization: Bearer $TOKEN"
```

Each call is listed under **Runs** in the editor. [Flows](../features/flows.md#test-publish-and-roll-back)
shows how to roll back to an earlier version.

## Store a value every flow can read

> **Included free** on every install.

A variable is a key and a value for your tenant. A node reads it as `{{ vars.<key> }}`.

<div class="paths">
<div data-path="console">

1. On **Flows**, choose **Variables**, then **New variable**.
2. Enter `site_url` in **Key** and your site's address in **Value**.
3. Tick **Secret** for a value nobody should read back, then choose **Create**.

A secret is stored encrypted, never shown again and never written to a run record.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/flows/variables \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key": "site_url", "value": "https://example.com", "is_secret": false}'
```

</div>
</div>

## Connect an outside system

> **Requires** a license with the `flow-pro` feature.

A datasource is a saved connection that nodes refer to by name: an outside PostgreSQL, MySQL
or SQL Server database, an HTTP API, or a Google service account.

<div class="paths">
<div data-path="console">

1. On **Flows**, choose **Datasources**, then **New datasource**.
2. Enter `partner_api` in **Name** and pick **HTTP base** in **Kind**.
3. Enter the API's address in **Base URL**, and pick its **Type** under **Authentication**.
4. Choose **Create**. The row's test button checks the connection.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/flows/datasources \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "partner_api", "kind": "http", "config": {"base_url": "https://api.example.com"}}'
```

</div>
</div>

On a free install, **Datasources** says datasources are not enabled on this engine, and
**New datasource** is unavailable.

![The Datasources page on a free install, with the not enabled notice.](./media/flows-datasources-free.png)

![The Datasources page on a licensed install, with partner_api tested.](./media/flows-datasources-licensed.png)

## Next

[14. Experiments, analytics and recommendations](experiments-and-analytics.md) tests two
versions of a trip page. For depth, read [Flows](../features/flows.md),
[Build a custom API with flows](../how-to/build-a-custom-api-with-flows.md) and
[Flow templates](../how-to/flow-templates.md).
