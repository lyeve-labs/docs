---
title: Flow Definition Format
description: "Look up any part of a flow definition: top-level keys, settings, graph rules, expressions, and every trigger and node with its settings and limits."
sidebar:
  order: 5
---

> **Included free** on every install. The types and options marked `Tier: pro` need a license with the `flow-pro` feature. See [pricing](https://lyeve.com/pricing).

A flow definition is the document a [flow](../features/flows.md) runs. The
canvas, the API, an exported file and the assistant all produce the same
document, so you can write one by hand, keep it in your repository and review
it like code. Use this page to look up a key, a rule or a node. To build a
first flow step by step, follow
[Build a custom API with flows](../how-to/build-a-custom-api-with-flows.md).

JSON is the canonical form and YAML is the same tree. Every example here is
YAML.

## Example

One `GET` endpoint that loads open orders with their customer, attaches each
order's shipments, caches the answer for 30 seconds and allows each caller 120
calls a minute:

```yaml
version: 1
name: Orders with customer and shipments
slug: orders-with-customer
description: One endpoint that joins three content types.
trigger:
  type: trigger.http
  config:
    method: GET
    auth: auth              # public | auth | admin
    rate_limit: { requests: 120, per: 1m }
    cache: { ttl: 30s }
settings:
  timeout: 10s              # whole run
  step_timeout: 5s
  max_steps: 200
  on_error: stop            # stop | continue
nodes:
  - id: orders
    type: content.query
    name: Load orders
    position: { x: 120, y: 200 }
    config:
      schema: orders
      filters: { status: "{{ trigger.query.status ?? 'open' }}" }
      populate: [customer]
      limit: 100
      sort: "-created_at"
  - id: shipments
    type: content.query
    position: { x: 420, y: 420 }
    config: { schema: shipments, filters: { order_id: "{{ map(input, .id) }}" } }
  - id: join
    type: data.join
    position: { x: 420, y: 300 }
    config: { left_key: id, right_key: order_id, how: left, as: shipments, many: true }
  - id: respond
    type: response
    position: { x: 720, y: 300 }
    config: { status: 200, body: "{{ input }}" }
edges:
  - { from: orders,    to: shipments }
  - { from: orders,    to: join,    to_port: left }
  - { from: shipments, to: join,    to_port: right }
  - { from: join,      to: respond }
notes:
  - { id: n1, text: "Cached 30s per URL and caller", position: { x: 700, y: 120 }, width: 220 }
```

## Top-level keys

| Key | Type | Meaning |
| --- | --- | --- |
| `version` | integer | Format version, always `1`. A missing one is read as `1` |
| `name` | string | Display name. An import without one uses the slug |
| `slug` | string | The flow's URL segment, `/api/v1/flows/{slug}`. Unique per tenant |
| `description` | string | Optional free text |
| `status` | `draft`, `active`, `disabled` | Written by an export. A created or imported flow always starts as `draft` |
| `trigger` | object | Exactly one: `type`, `config` and an optional `position` `{ x, y }` |
| `settings` | object | Run limits, below. Optional |
| `nodes` | list | Each with `id`, `type`, optional `name`, `position` `{ x, y }` and `config` |
| `edges` | list | Each with `from`, `to`, optional `from_port` and `to_port` |
| `notes` | list | Canvas annotations that never run: `id`, `text`, `position`, optional `width` and `height` |

An unknown top-level key is refused. An unknown key inside a node's `config`
is refused by that node type. An unknown key inside `settings`, the trigger, a
node, an edge or a note is dropped.

## Settings

| Key | Type | Meaning |
| --- | --- | --- |
| `timeout` | duration | Deadline for the whole run. Default `30s`, at most `5m` |
| `step_timeout` | duration | Deadline for each node, which also bounds a node's own `timeout`. SQL and HTTP calls inside the node are canceled with it. Default `10s`, at most `60s` |
| `max_steps` | integer | Most nodes one run may execute. Default 200, between 1 and 1000 |
| `on_error` | `stop`, `continue` | `stop` (default) fails the run at the first node error. `continue` marks the node failed, treats its outputs as inactive and keeps going, and the run finishes `succeeded` with the failed steps in its record |

A duration is a number with a unit: `500ms`, `30s`, `1m`, `1m30s`, `720h`. A
bare number is refused, so a missing unit is never read as nanoseconds.

## Rules

- A node `id` matches `[a-z][a-z0-9_]{0,39}` and is unique within the flow.
  `trigger` is reserved for the trigger. A `slug` matches
  `[a-z][a-z0-9_-]{0,39}`. A slug may carry hyphens and a node id may not,
  because an id is read in expressions as `nodes.<id>`.
- The trigger is the top-level `trigger` key. It is never an entry in
  `nodes`, and a node type is never used as the trigger.
- `from_port` defaults to `out` and `to_port` to `in`. An edge to a port the
  type does not declare fails validation. An edge cannot loop back to its own
  node, and nothing connects to the trigger or to a note.
- The graph is a directed acyclic graph. A cycle is a validation error.
  Fan-out is allowed. When several edges land on one port, the node receives
  their values as a list in edge order.
- A node with no inbound edge is a root and starts at once with the trigger
  payload as its input. An edge may also name `from: trigger` explicitly.
- A config expression may reference `nodes.<id>` only when `<id>` is an
  ancestor of the node through inbound edges, or the trigger. In the example,
  `shipments` reads the orders through `input` because an edge from `orders`
  lands on it. Without that edge the reference fails validation.
- `control.delay` is refused under `trigger.http`, because the caller is
  waiting.
- A node whose every inbound edge is inactive (its upstream branch did not
  fire) is skipped and marks its own ports inactive.
- With `on_error: continue`, a node that fails is recorded as failed, its
  ports are inactive, and the run still finishes `succeeded`. Read the steps,
  not only the run status.
- Positions are part of the definition, so an export imports again as drawn.
- A definition never carries a secret. Reference a datasource by name and a
  secret as `{{ vars.<key> }}`. An export drops any secret-named setting, such
  as a `password` or an `X-Api-Key` header, whose value is written out rather
  than read from a variable.
- A node type from another feature is valid only while that feature is
  licensed and running. A published flow whose version names a type that has
  gone is `blocked` until the type returns. Its URL then answers `503` with
  `flow is blocked: a node type it uses is not available on this instance`.

Saving, importing or publishing a definition that breaks a rule is refused
with `422`, and every problem names the node, the JSON pointer to the key, and
what is wrong. Two rules are checked only when you validate, test or publish:
the `nodes.<id>` ancestor rule and `control.delay` under `trigger.http`. A
draft that breaks either one still saves and imports.
`POST /api/admin/flows/{id}/validate` answers `200` with `ok: false` and the
same list instead of refusing. A refusal looks like this:

```json
{
  "ok": false,
  "errors": [
    { "node_id": "join", "path": "/config/left_key", "message": "required" }
  ]
}
```

A definition that uses a pro type or option without `flow-pro` is refused with
`402` and the same `errors` list. See
[free and flow-pro](../features/flows.md#free-and-flow-pro).

## Expressions

Any string in a config may carry `{{ ... }}` segments. The text between
segments is literal. A config value that is exactly one `{{ ... }}` segment
evaluates to the expression's value with its type kept: a list stays a list,
a map stays a map. Anything else, literal text around a segment or two
segments in one string, becomes a string. So `"{{ input }}"` hands the node
the upstream value as it is, and `"order {{ input.id }}"` hands it a string.

A config key whose type below is `expression` takes a bare expression with no
`{{ }}` around it. Every other string key may carry `{{ }}` segments.

An expression reads and never writes: it has no I/O and no assignment, and one
source is at most 64 KiB.

| Name | Value |
| --- | --- |
| `trigger` | The trigger envelope, below |
| `input` | The value on the node's default `in` port: the upstream node's output, or a list when several edges land |
| `inputs` | Every inbound port by name, each a list of the values that landed on it |
| `nodes` | `nodes.<id>.output` for every ancestor that has completed |
| `vars` | The tenant's variables, secrets included. A secret's value is masked in the run record |
| `run` | `{ id, flow, version, started_at, is_test }` |
| `item`, `index` | The current element and its position, inside `data.map`, `data.filter` and the body of a `control.foreach` |

The operators are the usual ones: `a.b`, `a?.b`, `a ?? b`, `==`, `&&`, `||`,
`in`, `contains`, `startsWith`, arithmetic, list literals and map literals
`{ key: value }`. Among the functions are `map`, `filter`, `first`, `last`,
`len`, `keys`, `values`, `get`, `join`, `split`, `lower`, `upper`, `trim`,
`now`, `fromJSON` and `toJSON`, plus `json`, which encodes a value as a JSON
string, and `uuid`, which returns a new UUID.

### Trigger envelope

Every trigger kind hands the flow the same keys, so an expression written for
an HTTP trigger evaluates to empty under any other kind instead of failing.

| Key | Value |
| --- | --- |
| `type` | The kind: `http`, `webhook`, `cron`, `event`, `manual` or `flow` |
| `method`, `path`, `ip` | Strings. Empty when the kind has no value for them |
| `query`, `params`, `headers` | Maps. Empty when the kind has no value for them |
| `body`, `user` | `null` when the kind has no value for them |

Each kind fills or adds:

| Trigger | Adds |
| --- | --- |
| `trigger.http` | `protocol` (`rest`, `graphql`, `grpc` or `realtime`), `method`, `path`, `query`, `params`, `headers`, `body`, `ip` and `user: { id, roles }`. `headers` never carries a credential header such as `Authorization`, `Cookie` or `X-API-Key`, because the payload is recorded with the run. `user` is `null` for a caller who is not signed in |
| `trigger.webhook` | The same keys as `trigger.http`, with the verified payload as `body` |
| `trigger.cron` | `scheduled_at` (RFC 3339) |
| `trigger.event` | `kind`, `name`, `schema`, `event`, `record_id`, `data` and `old_data`. For a content event, `name` and `event` are the change, such as `after_update`, and `data` is the entry. `old_data` is `null` on a create. For a system event, `name` and `event` are the event's name and `data` is its payload |
| `trigger.manual` | Whatever the run request supplied under `trigger` |
| `flow.call` (a parent flow) | `body` is the parent's input and `parent_run_id` is added |

## Import and export

`GET /api/admin/flows/{id}/export` downloads the draft as `<slug>.json`, or as
`<slug>.yaml` with `?format=yaml`, with the flow's current `status`.
`POST /api/admin/flows/import` reads a definition in one of three shapes, up to
4 MiB:

| Shape | Content type | Body |
| --- | --- | --- |
| The definition itself | `application/json` | The definition object |
| Text in an envelope | `application/json` | `{ "content": "<YAML or JSON text>", "format": "yaml" }`. `format` is `json`, `yaml` or empty to detect it |
| A file upload | `multipart/form-data` | A file part named `file`. The format is detected |

`?mode=create`, the default, creates a new flow and answers `201`.
`?mode=replace` replaces the draft of the flow with the same slug and answers
`200`. `?slug=` overrides the slug in the document. The answer carries
`unresolved_datasources`, the datasource names the definition uses that the
tenant does not have. Create them before the flow runs.
[Move flows between tenants and instances](../features/flows.md#move-flows-between-tenants-and-instances)
shows both calls.

## Datasources

A node that reaches an outside database, API or sheet names a datasource,
which holds the connection and its credential so the definition never does.
Creating one needs `flow-pro`. Datasources are managed at
`/api/admin/flows/datasources` by an `admin` or `super_admin`. The body is `name`, `kind`, `config` (returned
to admins) and `secret` (stored encrypted and never returned).

<details>
<summary>Settings for each datasource kind</summary>

| `kind` | `config` | `secret` |
| --- | --- | --- |
| `postgres`, `mysql`, `mssql` | `host`, `port`, `database`, `user`, `ssl`, `params` | `password` |
| `http` | `base_url`, `headers`, `auth`, and `chat_id` for a Telegram bot | `headers` for secret header values, and the `auth` secret below |
| `google_sheets` | `api_base` and `token_uri`, both optional | `service_account_json` |

An `http` datasource's `auth.type` picks the scheme:

| `auth.type` | `config.auth` keys | `secret` key |
| --- | --- | --- |
| `none` | | |
| `bearer` | | `token` |
| `basic` | `user` | `password` |
| `oauth2_client_credentials` | `token_url`, `client_id`, `scopes`, `audience` | `client_secret` |

`allow_writes` lets `db.query` commit, and `allow_private` lets the datasource
reach a private network address. Only a super admin may set either. A
datasource can never point at the instance's own database.

</details>

## Node types

Every node type declares its ports, a JSON Schema for its config, a
description, an example and a tier, `free` or `pro`. Your instance serves the
catalog at `GET /api/admin/flows/catalog` to anyone who may read at least one
flow. The editor's palette and inspector are drawn from it, and so is this
section. The same catalog as one Markdown document for a language model is
served at `GET /api/admin/flows/catalog/llm`, and a copy is published at
[The flow catalog for language models](flow-catalog-for-language-models.md).

A type marked `Tier: pro` saves, publishes and runs only with `flow-pro`. The
pro built-ins are:

- `trigger.webhook`
- `http.request`, `api.call`, `db.query`, `sheets.read`, `sheets.append`,
  `email.send` and `chat.notify`
- the `path` option of `trigger.http`, and every entry of its `protocols`
  other than `rest`
- any node whose config names a `datasource`, including one written as an
  expression

Everything else is free. Without `flow-pro`, a tenant holds at most twenty
flows.

A test run executes a node marked with a side effect as a dry run. It does
not act, and returns `{ dry_run: true, would: ... }` describing what it would
have done. A test request with `live: true` runs it for real.

### Editor hints

Some config keys carry hints for the admin console, all prefixed `x-`. They
change nothing about how a node runs, and a client that ignores them sends the
same config.

| Hint | Meaning |
| --- | --- |
| `x-source` | The console offers a picker: `content-schemas`, `content-fields`, `datasources` (with `x-kinds` naming the datasource kinds), `event-types` or `flows` |
| `x-keys-source` | The same, for the keys of a map |
| `x-editor` | `code` for a code editor, with `x-language` naming the language or `$<key>` to follow a sibling key, or `grid` for a table |
| `x-expression` | The key takes a bare expression |
| `x-duration` | The key takes a duration |
| `x-tier` | `pro` on an option that needs `flow-pro` when it is set |

### Nodes from other features

Other features add node and trigger types named `<feature>.<verb>`. They appear
in your catalog after the built-ins, with a `plugin` field naming the feature,
only where that feature is licensed and running, and each is documented on its
feature's page. A type from a free feature is free, except the email nodes and
`webhook.deliver`, which send outside the instance and need `flow-pro`. A type
from a licensed feature needs that feature and `flow-pro`.

| Type | What it does | Tier | Page |
| --- | --- | --- | --- |
| `captcha.verify` | Verify a captcha token | free | [Captcha](../features/captcha.md) |
| `error_tracking.capture` | Capture an error | free | [Error tracking](../features/error-tracking.md) |
| `localization.resolve` | Resolve a translation | free | [Localization](../features/localization.md#flows) |
| `review.transition` | Move a review to its next stage | free | [Editorial review](../features/editorial-review.md#in-a-flow) |
| `usage.quota_check` | Check a quota | free | [Usage and quotas](../features/usage-and-quotas.md) |
| `ai.complete`, `ai.classify`, `ai.extract`, `ai.image` | AI completion, classification, extraction and images | pro | [AI](../features/ai.md#use-ai-in-a-flow) |
| `audit.query`, `audit.record` | Query the audit log, record an entry | pro | [Audit log](../features/audit-log.md) |
| `data_export.start`, `data_export.status` | Start an export, read its status | pro | [Data export](../features/data-export.md) |
| `device_fingerprint.risk` | Score a device | pro | [Trusted devices](../features/trusted-devices.md) |
| `email.render`, `email.send_template` | Render an email template, send a templated email | pro | [Email](../features/email.md) |
| `events.publish` | Store an event | pro | [Event replay](../features/event-replay.md) |
| `events.received` (trigger) | Starts a flow when an event is stored | pro | [Event replay](../features/event-replay.md) |
| `graphql.query` | Run a GraphQL query | pro | [GraphQL API](../features/graphql-api.md) |
| `messagebroker.publish` | Publish a message | pro | [Message broker events](../features/message-broker-events.md) |
| `messagebroker.message` (trigger) | Starts a flow when a message arrives | pro | [Message broker events](../features/message-broker-events.md) |
| `realtime.publish` | Push a realtime event | pro | [Realtime](../features/realtime.md) |
| `webhook.deliver` | Deliver a webhook | pro | [Webhooks](../features/webhooks.md) |

Saving a definition that names a type this instance does not run fails
validation with
`node type "<type>" is not available on this instance; it needs the <plugin> plugin, which is not started`.

### System events

A `trigger.event` of kind `system` names an event the instance sends.
`GET /api/admin/flows/event-types` lists the content events and the system
events your instance can offer.

| Event | Sent when | Page |
| --- | --- | --- |
| `flow.run_failed` | A flow run ended failed | [Flows](../features/flows.md#get-told-when-a-flow-fails) |
| `review.transitioned` | A review assignment moved | [Editorial review](../features/editorial-review.md#events) |
| `review.sla.breached` | A review assignment passed its due time | [Editorial review](../features/editorial-review.md#events) |
| `cron.job.failed` | A scheduled job's run failed | [Scheduled jobs](../features/scheduled-jobs.md) |
| `cost.budget.exceeded` | A tenant's spend crossed a budget threshold | [Tenants](../features/tenants.md#tenant-costs) |
| `localization.translation.updated` | A translation was created, updated, deleted, imported or marked outdated | [Localization](../features/localization.md#flows) |

### Trigger

A flow has exactly one trigger. It is the top-level `trigger` key, not an entry in `nodes`, and it has no input port.

#### `trigger.cron`

Schedule. Starts the flow on a schedule. The schedule uses the five cron fields (minute, hour, day of month, month, day of week) with an optional leading seconds field, or a descriptor such as @hourly. Times are read in the timezone you name, UTC by default. The trigger payload carries scheduled_at. One run is in flight per flow at a time. A tick that lands while the previous run is still going waits, and runs once that run ends. Schedules are checked every 30 seconds, so a run can start up to 30 seconds after its time.

Inputs: none. Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `schedule` | string | yes |  | Cron expression, five fields with optional seconds, or a descriptor such as @hourly, @daily or @weekly. |
| `timezone` | string | no |  | IANA timezone name such as Europe/Amsterdam. Default UTC. |

Side effects: None.
Tier: free.

```yaml
config:
    schedule: 0 2 * * *
    timezone: UTC
```

#### `trigger.event`

Event. Starts the flow when an event is published on the instance. A content event fires after a record of a schema is created, updated or deleted. A system event fires when the instance sends the event you name, such as flow.run_failed. Records that content.upsert and content.delete write, and events that event.publish sends, never start an event flow. The flow runs asynchronously, so the write that caused it has already committed. The trigger payload carries kind, name, schema, event, record_id, data and, for content updates and deletes, old_data. For a system event data is the payload the publisher attached. Use * as the schema to react to every schema. The optional filter is evaluated against the trigger before a run starts, and a false result records nothing: no run is created and nothing is logged.

Inputs: none. Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `event` | one of `after_create`, `after_update`, `after_delete` | no |  | Which content change starts the flow. Required for kind content. |
| `filter` | expression | no |  | Evaluated against the trigger before the run starts. A false result starts nothing. For example trigger.data.status == 'paid'. |
| `kind` | one of `content`, `system` | no | `content` | What starts the flow: content is a record of a schema created, updated or deleted. System is a named event the instance sends, such as a failed flow run. |
| `name` | string | no |  | Name of the system event, such as flow.run_failed. Required for kind system. Any name of the shape [a-z][a-z0-9_.:-]{0,79} is accepted, except one that starts with before_ or after_. |
| `schema` | string | no |  | Content schema name, or * for every schema. Required for kind content. For a system event, optionally the schema the publisher names, such as a flow slug. |

Side effects: None.
Tier: free.

```yaml
config:
    event: after_create
    kind: content
    schema: customers
```

#### `trigger.http`

API request. Starts the flow when a caller reaches it and answers with whatever the response node sends, or the last output when there is none. REST reaches it at the flow's URL. With the flow-pro capability a GraphQL mutation, a gRPC call or a realtime socket message may start it too, each named in protocols. The trigger payload carries the protocol, method, path, query, params, headers (credential headers such as Authorization, Cookie and X-Api-Key are never included), body, client IP and the authenticated user. Choose who may call it, cap the body, and optionally rate limit and cache answers by a key expression.

Inputs: none. Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `auth` | one of `public`, `auth`, `admin` | no | `auth` | Who may call the flow: public answers anyone at the public URL, auth needs a signed-in user or API key, admin answers only on the editor's invoke route. |
| `body_limit` | integer | no |  | Largest request body in bytes. Default and at most 1 MiB. Set it lower to refuse large bodies sooner. |
| `cache` | object | no |  | Answer cache per key. |
| `cache.key` | expression | no |  | Expression that picks the cache entry, such as trigger.query.customer_id. Default is the request path and query. The surface and the signed-in user are always part of the entry. |
| `cache.ttl` | duration | yes |  | How long an answer is served from cache. |
| `method` | one of `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `ANY` | no | `ANY` | HTTP method the flow accepts. ANY accepts every method. |
| `path` | string | no |  | A URL of your own on the API server, such as /api/v1/orders/sync. It starts with /api/, is literal and unique across the instance, and cannot be under /api/admin or on a path the instance already serves, which always wins over it. Needs the flow-pro capability. Public REST flows only: nothing checks credentials at a path of your own. |
| `protocols` | list of one of `rest`, `graphql`, `grpc`, `realtime` | no | `[rest]` | Which protocols may start the flow. REST is the flow's URLs and is free. GraphQL (the runFlow mutation), gRPC (FlowService.Run) and realtime (a flow.run message on the socket) need the flow-pro capability. Default is REST only. |
| `rate_limit` | object | no |  | Token bucket per key. |
| `rate_limit.key` | expression | no |  | Expression that picks the bucket, such as trigger.ip. Default is the client IP. The surface and the signed-in user are always part of the bucket. |
| `rate_limit.per` | duration | yes |  | Window length, such as 1m. |
| `rate_limit.requests` | integer | yes |  | Requests allowed per window. |

Side effects: None.
Tier: free.

```yaml
config:
    auth: auth
    cache:
        key: trigger.query.customer_id
        ttl: 30s
    method: GET
    rate_limit:
        key: trigger.user?.id ?? trigger.ip
        per: 1m
        requests: 120
```

#### `trigger.manual`

Manual. Starts the flow only when someone allowed to run it does so from the editor or the admin API. The trigger payload is whatever the caller typed, so this is the trigger for one-off jobs and for flows you are still building.

Inputs: none. Outputs: `out`.

Config: none.

Side effects: None.
Tier: free.

#### `trigger.webhook`

Webhook. Starts the flow when an external system posts to the flow's hook URL. The sender signs the raw body with HMAC-SHA256 using the shared secret and sends the hex digest in the X-Flow-Signature header. A request with a missing or wrong signature is refused before the flow runs. Keep the secret in a secret variable and reference it with an expression, so an export never carries it.

Inputs: none. Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `body_limit` | integer | no |  | Largest request body in bytes. Default and at most 1 MiB. Set it lower to refuse large bodies sooner. |
| `path` | string | no |  | A URL of your own on the API server, such as /api/v1/orders/sync. It starts with /api/, is literal and unique across the instance, and cannot be under /api/admin or on a path the instance already serves, which always wins over it. Needs the flow-pro capability. The signature is checked there as on the hook URL. |
| `secret` | string | yes |  | Shared HMAC secret. Use {{ vars.<name> }} with a secret variable. |

Side effects: None.
Tier: pro. Needs the flow-pro capability.

```yaml
config:
    body_limit: 1048576
    secret: '{{ vars.github_hook_secret }}'
```

### Data

Nodes that read or write your content, the flow cache, a datasource or an outside service.

#### `cache.delete`

Cache clear. Removes one key, or every key under a prefix, from the flow cache, then passes the input on unchanged. Put it in an event flow to drop a cached answer when the record behind it changes: a prefix such as orders: clears every customer's list at once. Only entries of this tenant and flow are reachable.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `key` | string | no |  | One key to remove. Use this or prefix. |
| `prefix` | string | no |  | Remove every key that starts with this. |

Side effects: The entries are removed during a test run too.
Tier: free.

```yaml
config:
    prefix: 'orders:'
```

#### `cache.get`

Cache lookup. Looks a key up in the flow cache. On a hit the out port fires with the cached value. On a miss the miss port fires with the input unchanged, so the expensive path hangs off miss and ends in a cache.set with the same key. Keys are scoped to the tenant and the flow, so two flows never see each other's entries.

Inputs: `in` (accepts several edges). Outputs: `out`, `miss`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `key` | string | yes |  | Cache key. May carry {{ }} to scope it, such as customer:{{ trigger.query.id }}. |

Side effects: None.
Tier: free.

```yaml
config:
    key: orders:{{ trigger.query.customer_id }}
```

#### `cache.set`

Cache store. Stores a value in the flow cache under a key for a limited time, then passes the input on unchanged. Pair it with a cache.get on the same key: the miss branch computes, stores, and answers, and the next call is served from the hit. The lifetime defaults to one minute and is capped at a day. A test run stores nothing unless it is live.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `key` | string | yes |  | Cache key, the same shape the cache.get uses. |
| `ttl` | duration | no |  | How long the entry lives. Default 60s, at most 24h. |
| `value` | any | yes |  | What to store. {{ input }} stores the upstream output. |

Side effects: A test run reports what the node would do instead of doing it, unless the test is live.
Tier: free.

```yaml
config:
    key: orders:{{ trigger.query.customer_id }}
    ttl: 5m
    value: '{{ input }}'
```

#### `content.delete`

Delete record. Deletes one record of a content schema by id in the run's tenant. When the record existed, webhooks and other listeners receive after_delete, but no flow starts from it. A test run deletes nothing unless it is live, and the output reports what would have been. The output names the schema and the id.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `id` | string | yes |  | Record id. |
| `schema` | string | yes |  | Content schema to delete from. |

Side effects: A test run reports what the node would do instead of doing it, unless the test is live.
Tier: free.

```yaml
config:
    id: '{{ input.id }}'
    schema: sessions
```

#### `content.get`

Get record. Loads one record of a content schema by id in the run's tenant. The id usually comes from the trigger, such as trigger.params.id on an HTTP flow or trigger.record_id on an event flow. Populate attaches related records one level deep. A missing record gives a null output, so put a condition after it when the record may be absent.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `id` | string | yes |  | Record id. |
| `populate` | list of strings | no |  | Relation fields to resolve, one level deep. |
| `schema` | string | yes |  | Content schema to read. |

Side effects: None.
Tier: free.

```yaml
config:
    id: '{{ trigger.params.id }}'
    populate:
        - account
    schema: customers
```

#### `content.query`

Query content. Lists records of a content schema in the run's tenant. Filters match by equality, or by membership when the value is a list, so a filter built from the previous node's ids loads every related row in one step. Populate attaches related records one level deep. Sort with a field name, or a leading minus for descending. At most 1000 rows per node. Page with offset for more.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `fields` | list of strings | no |  | Fields to keep on each row. Empty keeps every field. |
| `filters` | object | no |  | Field to value. A list value matches any of its entries. |
| `limit` | integer | no |  | Rows to return. Default 100, at most 1000. |
| `offset` | integer | no |  | Rows to skip. |
| `populate` | list of strings | no |  | Relation fields to resolve, one level deep. |
| `schema` | string | yes |  | Content schema to read. |
| `sort` | string | no |  | Field to order by. Prefix it with a minus sign for descending order. |

Side effects: None.
Tier: free.

```yaml
config:
    filters:
        status: '{{ trigger.query.status ?? ''open'' }}'
    limit: 100
    populate:
        - customer
    schema: orders
    sort: -created_at
```

#### `content.upsert`

Write record. Creates or updates one record of a content schema in the run's tenant. With an id the record is updated, or created under that id. Without one a new record is created. An update takes the data as a patch: fields it does not name keep their stored value. The document is validated against the schema. After the write, webhooks and other listeners receive after_create or after_update, but no flow starts from it. No revision is kept. A test run writes nothing unless it is live, and the output reports what would have been. The output is the record as stored.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `data` | object | yes |  | Fields to store. On an update, the fields to change. {{ input }} writes the upstream object. |
| `id` | string | no |  | Record id to update. Leave empty to create. |
| `schema` | string | yes |  | Content schema to write. |

Side effects: A test run reports what the node would do instead of doing it, unless the test is live.
Tier: free.

```yaml
config:
    data:
        company: '{{ input.body.name }}'
        enriched_at: '{{ now() }}'
    id: '{{ trigger.record_id }}'
    schema: customers
```

#### `data.inline`

Inline data. Emits data typed into the flow itself: a JSON or YAML document, CSV rows, plain text, or a small table of columns and rows. Use it for lookup tables, fixtures and test input that would otherwise need a datasource. JSON and YAML parse to their value, CSV with a header gives a list of objects and without one a list of lists, with every cell kept as a string. The content may carry {{ }}, which is evaluated before parsing. At most 1 MiB of content, or 64 KiB when it carries {{ }}.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `columns` | list of strings | no |  | Table only: column names, in order. |
| `content` | string | no |  | The data, in the chosen format. Not used for table. |
| `format` | one of `json`, `yaml`, `csv`, `text`, `table` | yes |  | How to read the content. |
| `header` | boolean | no | `true` | CSV only: the first row names the columns and rows become objects. |
| `rows` | list of list of anys | no |  | Table only: one list of cells per row, as long as columns. |

Side effects: None.
Tier: free.

```yaml
config:
    content: |-
        sku,qty,price
        A-100,2,9.99
        B-200,1,24.50
        C-300,5,3.25
    format: csv
    header: true
```

#### `db.query`

SQL query. Runs a SQL statement against a stored Postgres, MySQL or SQL Server datasource. Write placeholders as $1, $2 on every database and list one expression per placeholder in params. The values are bound, never spliced into the text. The statement runs inside a transaction that is rolled back unless the datasource allows writes and commit is on. A test run commits only when it is live. The output is the list of rows as objects.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `commit` | boolean | no |  | Commit the transaction. Needs a datasource that allows writes. |
| `datasource` | string | yes |  | Name of a SQL datasource. |
| `max_rows` | integer | no |  | Rows to return at most. Default 500, at most 5000. |
| `params` | list of expressions | no |  | One expression per placeholder, in order. |
| `sql` | string | yes |  | The statement, with positional placeholders. |

Side effects: A test run that is not live never commits: `commit` is ignored and the transaction is rolled back.
Tier: pro. Needs the flow-pro capability.

```yaml
config:
    datasource: warehouse
    max_rows: 500
    params:
        - trigger.query.customer_id
        - '''paid'''
    sql: SELECT id, total FROM orders WHERE customer_id = $1 AND status = $2
```

#### `email.send`

Send email. Sends an email through the sender the instance is configured with. Recipients are a list or one comma-separated string, and the message needs a text body or an HTML body. When both are set, only the text body is sent. The node fails with a clear message when the instance has no sender, so a flow does not silently drop mail. A test run sends nothing unless it is live, and the output reports the message that would have gone out.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `html` | string | no |  | HTML body. |
| `subject` | string | yes |  | Subject line. May carry {{ }}. |
| `text` | string | no |  | Plain-text body. |
| `to` | list of strings | yes |  | Recipients. A comma-separated string is accepted too. |

Side effects: A test run reports what the node would do instead of doing it, unless the test is live.
Tier: pro. Needs the flow-pro capability.

```yaml
config:
    subject: '{{ len(input) }} orders need attention'
    text: 'Open orders older than a day: {{ json(input) }}'
    to:
        - ops@example.com
```

#### `event.publish`

Publish event. Publishes a content event as if the record had changed, so webhooks and message brokers subscribed to that schema and event react to it. Flows do not start from it. Use it to fan a flow's result out to every listener without wiring each one. The event carries the flow as its source. A test run publishes nothing unless it is live.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `data` | object | no |  | Record data the listeners receive. {{ input }} sends the upstream object. |
| `event` | one of `after_create`, `after_update`, `after_delete` | yes |  | Which change to announce. |
| `record_id` | string | yes |  | Id of the record the event is about. |
| `schema` | string | yes |  | Content schema the event is about. |

Side effects: A test run reports what the node would do instead of doing it, unless the test is live.
Tier: free.

```yaml
config:
    data: '{{ input }}'
    event: after_update
    record_id: '{{ input.id }}'
    schema: orders
```

#### `http.request`

HTTP request. Calls an external HTTP API. Private network addresses are refused, unless the node names a datasource that allows them. Give a full url, or name an HTTP datasource and a path under its base. The datasource's stored headers apply, with the node's own headers winning per name, and its auth (a bearer token, basic credentials or an OAuth2 client-credentials token, fetched and cached by the instance) applies unless the node sets Authorization itself. A path cannot climb above the base. A JSON body is encoded and a JSON answer decoded, so input.body.name reads straight from the response. GET and HEAD run for real during a test. Any other method is only reported, unless the test is live. The output is { status, headers, body }.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `body` | any | no |  | Request body. An object or list is sent as JSON, a string as is. |
| `datasource` | string | no |  | Name of an HTTP datasource whose base URL and headers apply. |
| `headers` | map of strings | no |  | Request headers. They override the datasource's. |
| `method` | one of `GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE` | no |  | HTTP method. Default GET. |
| `path` | string | no |  | Path under the datasource base, such as /v1/customers. |
| `query` | map of strings | no |  | Query parameters added to the URL. |
| `retries` | integer | no |  | Attempts after a network failure, a 429 or a 5xx answer. At most 3. |
| `timeout` | duration | no |  | Time allowed for each attempt. Default 10s, at most 30s. The step timeout still applies. |
| `url` | string | no |  | Full URL to call. Use this or datasource. |

Side effects: A test run performs `GET` and `HEAD` and reports any other method as a dry run, unless the test is live.
Tier: pro. Needs the flow-pro capability.

```yaml
config:
    headers:
        Accept: application/json
        Authorization: Bearer {{ vars.api_key }}
    method: GET
    retries: 2
    timeout: 10s
    url: https://api.example.com/v1/domains/{{ split(trigger.data.email, '@')[1] }}
```

#### `sheets.append`

Append to sheet. Appends rows below the last filled row of a range in a Google Sheet, through a google_sheets datasource. Rows may be lists of cell values, written as given, or objects, whose values are written in the column order of the first object's keys sorted by name. Use data.pick upstream to choose the columns. A test run writes nothing unless it is live, and the output reports how many rows would have gone where.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `datasource` | string | yes |  | Name of a google_sheets datasource. |
| `range` | string | yes |  | A1 notation naming the table, such as Orders!A1. |
| `rows` | list of anys | yes |  | Rows to append: lists of cells, or objects. {{ input }} appends the upstream list. |
| `spreadsheet_id` | string | yes |  | The id from the sheet's URL. |

Side effects: A test run reports what the node would do instead of doing it, unless the test is live.
Tier: pro. Needs the flow-pro capability.

```yaml
config:
    datasource: reports
    range: Orders!A1
    rows: '{{ input }}'
    spreadsheet_id: '{{ vars.orders_sheet_id }}'
```

#### `sheets.read`

Read sheet. Reads a range of a Google Sheet through a google_sheets datasource, which holds the service account the sheet is shared with. Without header the output is a list of rows, each a list of cell values. With header the first row names the columns and every other row becomes an object keyed by those names, which is what the transform nodes expect.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `datasource` | string | yes |  | Name of a google_sheets datasource. |
| `header` | boolean | no |  | Treat the first row as column names and emit objects. |
| `range` | string | yes |  | A1 notation, such as Orders!A1:F. |
| `spreadsheet_id` | string | yes |  | The id from the sheet's URL. |

Side effects: None.
Tier: pro. Needs the flow-pro capability.

```yaml
config:
    datasource: reports
    header: true
    range: Orders!A1:F
    spreadsheet_id: '{{ vars.orders_sheet_id }}'
```

### Integration

Nodes that reach another system: the instance's own API, another flow, or a chat service, so a flow composes what already exists instead of copying it.

#### `api.call`

Call own API. Calls this instance's own API: content, every other API route and other flows, under that route's own permission, license and rate checks. Give a path under /api/. The admin surface is refused. The run's tenant travels with the call. Auth caller forwards the Authorization header, API key and cookies of the request that started the run, so the call acts as that user (HTTP triggers only). Auth variable sends a secret variable as a bearer token or an API key, which is how a scheduled or event flow calls a protected route. A JSON body is encoded and a JSON answer decoded. The output is { status, headers, body }. A non-2xx answer fails the node unless allow_failure is set. Calls that reach a flow again stop at five hops.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `allow_failure` | boolean | no |  | Return a non-2xx answer as output instead of failing the node. |
| `auth` | one of `none`, `caller`, `variable` | no | `none` | Whose credential the call carries: none, caller (the Authorization header, API key and cookies of the HTTP request that started the run) or variable (a secret variable). |
| `body` | any | no |  | Request body. An object or list is sent as JSON, a string as is. |
| `headers` | map of strings | no |  | Extra request headers. Authorization, Cookie, X-API-Key, X-Tenant-ID and X-CSRF-Token are refused here, because auth sets them. |
| `method` | one of `GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE` | no |  | HTTP method. Default GET. |
| `path` | string | yes |  | Path on this instance's API, such as /api/v1/content/orders. Must begin with /api/ and never /api/admin. |
| `query` | map of strings | no |  | Query parameters added to the URL. |
| `timeout` | duration | no |  | Time allowed for the call. Default 10s, at most 30s. |
| `token_header` | one of `bearer`, `api_key` | no |  | How the variable is sent: bearer puts it in Authorization as a Bearer token, api_key in X-API-Key. Default bearer. |
| `variable` | string | no |  | Name of the secret variable holding the token, for auth variable. |

Side effects: A test run reports the method, path and auth mode instead of calling, unless the test is live.
Tier: pro. Needs the flow-pro capability.

```yaml
config:
    auth: caller
    method: GET
    path: /api/v1/content/orders
    query:
        limit: "20"
        sort: -created_at
```

#### `chat.notify`

Chat notification. Posts a message to Slack, Discord or Telegram in that service's own format. The datasource is an HTTP datasource: for Slack and Discord its base URL is the incoming webhook URL. For Telegram it is the Bot API base with the token, https://api.telegram.org/bot<token>, and the chat comes from chat_id on the node or on the datasource. The title and each field are rendered literally, so a record value cannot open formatting or mention a group. The text is sent as written, escaped for Slack and Telegram, and Discord disables every mention. A test run sends nothing unless it is live. The output is { channel, status, message_id }, the id empty when the service returns none.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `channel` | one of `slack`, `discord`, `telegram` | yes |  | Which service the datasource points at. |
| `chat_id` | string | no |  | Telegram chat id or @channel. Overrides the datasource's chat_id. |
| `datasource` | string | yes |  | HTTP datasource carrying the webhook URL, or the Bot API base with the token. |
| `fields` | map of strings | no |  | Named values shown as attachment fields where the service has them, else as lines. |
| `text` | string | no |  | Body of the message. May carry {{ }}. |
| `timeout` | duration | no |  | Time allowed for the call. Default 10s, at most 30s. |
| `title` | string | yes |  | Heading of the message, rendered literally. May carry {{ }}. |

Side effects: A test run reports what the node would do instead of doing it, unless the test is live.
Tier: pro. Needs the flow-pro capability.

```yaml
config:
    channel: slack
    datasource: chat
    fields:
        id: '{{ trigger.record_id }}'
        status: '{{ trigger.data.status }}'
    text: '{{ trigger.data.title }} was saved by {{ trigger.data.author }}'
    title: '{{ trigger.schema }} {{ trigger.event }}'
```

#### `flow.call`

Call flow. Runs another published flow of this tenant and hands its answer on. The child sees the input as trigger.body with trigger.type set to flow, runs in this run's tenant, within this node's step timeout when it waits, and is recorded as its own run linked to this one. Use it to keep a shared step, such as an enrichment or a notification, in one flow that many others call. A flow cannot call itself, directly or through a chain, and calls stop five deep. The output is the child's response body or, when it has no response node, its last node's output. With wait false the child is started and only { run_id, status: running } comes back.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `flow` | string | yes |  | Slug of a published flow of this tenant. |
| `input` | any | no |  | What the child sees as trigger.body. {{ input }} passes this node's input on. |
| `wait` | boolean | no |  | Wait for the child and return its answer. Default true. |

Side effects: A test run reports the flow and input instead of running it, unless the test is live.
Tier: free.

```yaml
config:
    flow: enrich-customer
    input: '{{ input }}'
    wait: true
```

### Transform

Pure reshaping of what arrived. Nothing leaves the run.

#### `data.aggregate`

Aggregate. Reduces a list of objects to one row per group with the metrics you name: count, sum, avg, min or max of a field. Without group_by the output is a list holding one row over the whole list. Values that are not numbers are ignored by sum, avg, min and max, and a group with none of them gets null.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `group_by` | string | no |  | Field whose distinct values form the groups. Leave empty for one row. |
| `metrics` | list of objects | yes |  | One output field per entry. |
| `metrics[].field` | string | no |  | Field to reduce. Not needed for count. |
| `metrics[].fn` | one of `count`, `sum`, `avg`, `min`, `max` | yes |  | How to reduce the group. |
| `metrics[].name` | string | yes |  | Output field name. |

Side effects: None.
Tier: free.

```yaml
config:
    group_by: status
    metrics:
        - fn: count
          name: orders
        - field: total
          fn: sum
          name: revenue
```

#### `data.filter`

Filter. Keeps the items of a list for which an expression is true. Inside the expression item is the current element and index its position. Empty strings, zero, null, false and empty collections count as false. Everything else keeps the item.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `expression` | expression | yes |  | Evaluated once per element with item and index bound. A truthy result keeps it. |

Side effects: None.
Tier: free.

```yaml
config:
    expression: item.status == 'open' && item.total > 100
```

#### `data.join`

Join. Joins two lists of objects on a key, the way a SQL join does. Every left row gets the matching right rows under the name you choose: one object when many is off, a list when it is on. An inner join drops left rows with no match. A left join keeps them with null or an empty list. Three tables are two joins in a row. A join whose output would exceed 100000 rows, nested matches counted, is refused.

Inputs: `left` (accepts several edges), `right` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `as` | string | yes |  | Field name that receives the match on each left row. |
| `how` | one of `inner`, `left` | no | `inner` | inner keeps only matched left rows. Left keeps them all. |
| `left_key` | string | yes |  | Field on each left row to match. Dots reach into nested objects. |
| `many` | boolean | no |  | Attach every match as a list instead of the first one. |
| `right_key` | string | yes |  | Field on each right row to match. |

Side effects: None.
Tier: free.

```yaml
config:
    as: shipments
    how: left
    left_key: id
    many: true
    right_key: order_id
```

#### `data.map`

Map. Rebuilds every item of a list with an expression. Inside the expression item is the current element and index its position, and the rest of the environment is there too, so a row can be reshaped, renamed or combined with a value from an earlier node. The output is a list of the same length.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `expression` | expression | yes |  | Evaluated once per element with item and index bound. |

Side effects: None.
Tier: free.

```yaml
config:
    expression: '{ id: item.id, total: item.qty * item.price }'
```

#### `data.pick`

Pick fields. Keeps only the named fields of every object in a list. A field an item does not have is left out rather than set to null. Use it before a response or an export to send exactly the columns a consumer expects.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `fields` | list of strings | yes |  | Field names to keep on each item. |

Side effects: None.
Tier: free.

```yaml
config:
    fields:
        - id
        - email
        - created_at
```

#### `data.set`

Set value. Emits a value you compose: a literal, an object whose fields carry expressions, or a single expression over the input and earlier nodes. It is the simplest way to compute something once and hand it on, or to shape the exact payload another node expects.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `value` | any | yes |  | The value to emit. Strings may carry {{ }}. A value that is exactly one {{ }} keeps its type. |

Side effects: None.
Tier: free.

```yaml
config:
    value:
        count: '{{ len(input) }}'
        customer_id: '{{ trigger.query.customer_id }}'
```

### Control

Routing and iteration.

#### `control.condition`

Condition. Sends the input down one of two paths. When the expression is true the true port fires, otherwise the false port. The other side stays inactive and every node that only hangs off it is skipped. Both ports carry the input unchanged.

Inputs: `in` (accepts several edges). Outputs: `true`, `false`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `expression` | expression | yes |  | Evaluated once against the input. A truthy result fires true. |

Side effects: None.
Tier: free.

```yaml
config:
    expression: len(input) > 0
```

#### `control.delay`

Delay. Waits for a duration, then passes the input on unchanged. It is refused on a flow with an HTTP trigger because the caller would be kept waiting. Use it in scheduled and event flows, for example to give an external system time to settle before reading back. At most 30 seconds, and the step timeout still applies.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `duration` | duration | yes |  | How long to wait, such as 5s. At most 30s. |

Side effects: None.
Tier: free.

```yaml
config:
    duration: 5s
```

#### `control.fail`

Fail. Ends the run as failed with a message, whatever the on_error setting says. On an HTTP trigger the caller receives the status you choose with the message as the error. Put it behind a condition to refuse input the flow cannot handle.

Inputs: `in` (accepts several edges). Outputs: none.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `message` | string | no |  | Why the run failed. Recorded on the run and returned to an HTTP caller. |
| `status` | integer | no |  | HTTP status for an HTTP caller. Default 500. |

Side effects: None.
Tier: free.

```yaml
config:
    message: customer {{ trigger.query.customer_id }} has no orders
    status: 404
```

#### `control.foreach`

For each. Runs the nodes connected to its item port once per element of the input list, one element at a time, with item and index available to their expressions. When every element has been through, the done port fires with a list holding the last body node's output for each element, in order. Nodes that should run once, after the loop, connect to done.

Inputs: `in` (accepts several edges). Outputs: `item`, `done`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `max_items` | integer | no |  | Refuse a list longer than this. Default and cap 1000. |

Side effects: None.
Tier: free.

```yaml
config:
    max_items: 200
```

#### `control.parallel`

Parallel. Starts the nodes connected to its out port at the same time instead of one after another. Nodes further down a branch run one after another, as anywhere else in the flow. Each branch gets the input. A node that joins several branches waits for all of them. The first branch that fails stops the others when the flow stops on error. Use it when two lookups do not depend on each other. Content reads and writes in different branches run one at a time. External datasources and HTTP calls run side by side.

Inputs: `in` (accepts several edges). Outputs: `out`.

Config: none.

Side effects: None.
Tier: free.

#### `control.switch`

Switch. Routes the input to the first case whose expression is true. Each case names an output port and the port appears on the node as soon as the case exists. When no case matches, the default port fires. Cases are tried in the order listed, so put the most specific first.

Inputs: `in` (accepts several edges). Outputs: `default`.
The config adds output ports of its own. See the table.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `cases` | list of objects | yes |  | Tried in order. The first truthy when wins. |
| `cases[].port` | string | yes |  | Output port name for this case. |
| `cases[].when` | expression | yes |  | Condition evaluated against the input. |

Side effects: None.
Tier: free.

```yaml
config:
    cases:
        - port: eu
          when: trigger.query.region == 'eu'
        - port: us
          when: trigger.query.region == 'us'
```

### Output

How a run ends or leaves a trace.

#### `log`

Log. Writes a line to the run's step record and to the server log, then passes the input on unchanged. The message may carry expressions, and fields adds structured values next to it. Put one after a node you are debugging to see what it produced without stopping the run.

Inputs: `in` (accepts several edges). Outputs: `out`.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `fields` | object | no |  | Extra values to log alongside the message. |
| `level` | one of `debug`, `info`, `warn`, `error` | no |  | Severity. Default info. |
| `message` | string | yes |  | What to log. May carry {{ }}. |

Side effects: None.
Tier: free.

```yaml
config:
    fields:
        customer: '{{ trigger.query.customer_id }}'
    level: info
    message: loaded {{ len(input) }} orders
```

#### `note`

Note. A sticky note on the canvas for the people who read the flow. It has no ports, takes no edges and never runs. It exists so the reasoning behind a branch or a cache key travels with the export.

Inputs: none. Outputs: none.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `text` | string | no |  | What the note says. |
| `width` | integer | no |  | Width on the canvas in pixels. |

Side effects: None.
Tier: free.

```yaml
config:
    text: Cached 30s per customer. Clear it with cache.delete on order update.
    width: 220
```

#### `response`

Response. Ends the run and, on an HTTP trigger, answers the caller with this status, these headers and this body. A body that is exactly one expression keeps its type, so {{ input }} returns the upstream list as JSON. Nothing after this node runs. A flow with an HTTP trigger and no response answers 200 with the last output.

Inputs: `in` (accepts several edges). Outputs: none.

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `body` | any | no |  | Response body. {{ input }} sends the upstream output. |
| `headers` | map of strings | no |  | Response headers. |
| `status` | integer | no |  | HTTP status. Default 200. |

Side effects: None.
Tier: free.

```yaml
config:
    body: '{{ input }}'
    headers:
        Cache-Control: private, max-age=30
    status: 200
```

## Related

- [Flows](../features/flows.md): publishing, permissions, URLs and the free tier.
- [Build a custom API with flows](../how-to/build-a-custom-api-with-flows.md):
  the example above, built step by step.
- [The flow catalog for language models](flow-catalog-for-language-models.md):
  this format as one document to give a model.
- [Data model](../concepts/data-model.md#relations): how relation fields, such
  as `customer_id`, are declared.
