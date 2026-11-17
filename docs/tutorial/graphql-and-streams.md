---
title: "11. GraphQL, gRPC, streams and realtime"
description: "Send real requests from API labs, query trips over GraphQL, register a persisted query, check the gRPC listener, watch content events arrive on a stream, and see the message broker's state."
sidebar:
  order: 11
covers:
  console:
    - "/admin/api-labs"
    - "/admin/api-labs/graphql"
    - "/admin/api-labs/grpc"
    - "/admin/api-labs/streams"
    - "/admin/settings/graphql"
    - "/admin/settings/graphql/persisted"
    - "/admin/settings/grpc"
    - "/admin/realtime"
    - "/admin/settings/message-broker"
  api:
    - "GET /api/admin/openapi.json"
    - "GET /api/v1/content/{schema}"
    - "POST /api/v1/graphql"
    - "POST /api/admin/graphql/persisted-queries"
    - "GET /api/admin/grpc/status"
    - "GET /api/v1/realtime/events"
    - "GET /api/admin/realtime/metrics"
    - "GET /api/admin/messagebroker/status"
verified:
  engine: v0.62.0
  console: v0.28.0
---

The REST Content API is one way to read your content. In this lesson you send requests from
**API labs**, read the `trip` entries over GraphQL and gRPC, and watch a change arrive on a
live stream. You need the `trip` content type from [lesson 3](model-content.md), a few trips from
[lesson 4](write-and-publish.md), and a token in `TOKEN` from [lesson 10](api-access.md).

Every request in **API labs** goes to the engine you are signed in to, as you. There is no
sandbox, so a write changes data and a delete removes it.

## Send a request from API labs

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Open **Delivery > API labs**. The **REST** tab lists every endpoint this engine serves.
2. Type `content` in the search box and choose **Select** on the row for
   `GET /api/v1/content/{schema}`.
3. Enter `trip` in **schema (in the path)** and choose **Send**.

The answer shows the status and the body, with the trips you wrote in lesson 4.

</div>
<div data-path="api">

```bash
curl http://localhost:3002/api/v1/content/trip \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![Searching the endpoint list in API labs, selecting the content list route, entering trip and sending it.](./media/graphql-and-streams-rest-send.mp4)

The list comes from the engine's own API document, so a route a feature adds appears here
while that feature runs. When the document cannot be read, the page says so and falls back
to the catalog built into the console.

## Query trips over GraphQL

> **Requires** a license with the `graphql` feature.

GraphQL builds its schema from your content types, so `trip` already has a `trip` query and a
`createTrip` mutation.

<div class="paths">
<div data-path="console">

1. Open **Delivery > GraphQL**. **Endpoints** gives the address for queries and mutations,
   and the one for subscriptions. **Schema** lists every field built from your content types.
2. Choose **Try it in API labs**, which opens the **GraphQL** tab.
3. Enter this in **Query or mutation** and choose **Send**:

   ```graphql
   { trip(limit: 5) { id title } }
   ```

GraphQL answers `200` even when a query fails, so read the body. The lab warns you when a
`200` answer carries an `errors` array.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3002/api/v1/graphql \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ trip(limit: 5) { id title } }"}'
```

</div>
</div>

On an install without the `graphql` feature, **GraphQL** shows a card that says it is not
enabled on this engine, in place of the endpoints and the schema.

![The GraphQL page on a free install, with the not enabled card.](./media/graphql-and-streams-graphql-free.png)

![The GraphQL page on a licensed install, with the endpoints and the schema built from trip.](./media/graphql-and-streams-graphql-licensed.png)

## Register a persisted query

> **Requires** a license with the `graphql` feature.

A client can send the hash of a query instead of its text, and only queries on this list run
by hash.

<div class="paths">
<div data-path="console">

1. On **Delivery > GraphQL**, open the **Persisted queries** tab.
2. Choose **New persisted query**.
3. Enter `{ trip(limit: 10) { id title } }` in **Query**, `LatestTrips` in
   **Operation name** and a line in **Description**, then choose **Create**.

The query is listed with its **Hash**. The row's buttons copy the hash, disable the query and
delete it. A disabled query is refused until you enable it again.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/graphql/persisted-queries \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ trip(limit: 10) { id title } }", "operation_name": "LatestTrips"}'
```

The answer is `201` with `query_hash`, the SHA-256 of the query text.

</div>
</div>

![Opening New persisted query, entering the trip query and its operation name, and creating it.](./media/graphql-and-streams-persisted-query.mp4)

[GraphQL API](../features/graphql-api.md#lock-clients-to-persisted-queries) shows how a client
sends the hash, and how to refuse every query that is not on the list.

## Check the gRPC listener

> **Requires** a license with the `grpc` feature.

Only a super admin sees this screen, because it shows the addresses the engine listens on.

<div class="paths">
<div data-path="console">

Open **Delivery > gRPC**. **Listeners** gives the **gRPC** address and the **Health and REST**
address, and whether **Reflection** is on. **Services** lists each service with its health
status. **REST, without a gRPC client** lists the JSON routes the health listener serves.

To call one, open **Delivery > API labs**, choose the **gRPC** tab, pick
`GET /api/content/{schema}` in **Transcoded route**, enter `trip` and choose **Run**.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/grpc/status \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

When the feature runs but no listener started, the page says to look for a bind failure in
the engine's start-up log. The listeners are set in the engine's environment, as
[gRPC API](../features/grpc-api.md#try-it) shows.

![The gRPC page with both listeners, reflection, and the services each reported as serving.](./media/graphql-and-streams-grpc.png)

## Watch content events arrive

> **Requires** a license with the `realtime` feature.

<div class="paths">
<div data-path="console">

1. Open **Delivery > API labs** and choose the **Streams** tab.
2. Choose **Server-sent events** in **Transport** and enter `content:trip` in **Topics**.
3. Choose **Connect**.
4. In a second browser tab, open a trip and save it.

The event appears under **What arrives** with its time, its topic and the payload, which
holds the record and the action. **Disconnect** closes the stream and **Clear** empties the
list.

</div>
<div data-path="api">

Open the stream and leave it running:

```bash
curl -N "http://localhost:3002/api/v1/realtime/events?topic=content:trip" \
  -H "Origin: http://localhost:3002" \
  -H "Authorization: Bearer $TOKEN"
```

Save a trip in another terminal or in the console. The stream prints an `event: content:trip`
block with the record id and the action.

</div>
</div>

![Connecting a stream on content:trip, saving a trip in a second tab, and the event arriving in the list.](./media/graphql-and-streams-stream.mp4)

A refused stream usually means the feature is not running, or that the page's origin is not
the engine's own host or one of its allowed origins.

## See who is connected

> **Requires** a license with the `realtime` feature.

<div class="paths">
<div data-path="console">

Open **Insight > Realtime**. **Server-sent events** and **WebSocket** count the connections
**Open now** for your tenant, and the events pushed, broadcasts and rejections across the
whole engine. **Connections by tenant** lists each tenant holding a stream open. Choose
**Refresh** to read the counts again.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/realtime/metrics \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![The Realtime page with an open stream counted under server-sent events.](./media/graphql-and-streams-realtime.png)

## Check the message broker

> **Requires** a license with the `messagebroker` feature.

The message broker publishes every content create, update and delete to NATS, Kafka or
RabbitMQ. Only a super admin sees this screen, because it covers every tenant.

<div class="paths">
<div data-path="console">

Open **Settings > Message broker**. The counters show what was **Published**, what had
**Nowhere to send**, what is **Waiting in the outbox** and what was
**Lost after every attempt** since the engine last started. **Connection** names the
**Backend**, the **Target** and whether **TLS** is on. When no broker is set, the page says
that events are discarded and names the setting to change.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/messagebroker/status \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

The broker is set in the engine's environment, never on this page.
[Message broker events](../features/message-broker-events.md#settings) lists the settings.

![The Message broker page with its counters and a connected broker.](./media/graphql-and-streams-message-broker.png)

## Next

[12. Webhooks, events and email](webhooks-and-email.md) sends a trip change to another
service. For depth, read [GraphQL API](../features/graphql-api.md),
[gRPC API](../features/grpc-api.md), [Realtime](../features/realtime.md) and
[Message broker events](../features/message-broker-events.md).
