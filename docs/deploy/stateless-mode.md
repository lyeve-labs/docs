---
title: Stateless mode
description: "Run the engine with no database as a webhook relay, with flows, webhooks and API keys declared in the configuration file."
sidebar:
  order: 14
---

> **Included free** on every install, with up to twenty flows and 25 outbound webhooks. A relay's
> webhook trigger, custom paths, outbound nodes and more flows require a license with the
> `flow-pro` feature, and more webhooks, payload templates, filters and retry settings the
> `webhook-pro` feature. See [pricing](https://lyeve.com/pricing/).

`LYEVE_MODE=stateless` starts the engine with no database at all. It receives webhooks,
reshapes them with a flow and passes them on, all in one container, with no database to run or
pay for. Everything it would keep in a database comes from the configuration file.

:::tip[Best for]
- **A webhook relay.** Receive a Stripe or GitHub webhook, check its signature, and forward a
  reshaped payload to your CRM.
- **An API on the edge of your network** that answers from flows and calls other systems, with
  nothing to back up.
- **Disposable environments** built from one file in your repository.
:::

:::caution[What you give up]
- **Nothing survives a restart.** Run history, pending webhook retries and recent deliveries
  live in memory. A retry still pending when the container stops is lost, and the count is
  logged at shutdown.
- **Nothing is metered.** Usage is not counted, and an API key has no monthly limit.
- **One tenant.** Every request runs as the `default` tenant.
- **No admin console.** There is no sign-in and no Admin API listener. You read status through
  the API with a key declared in the file.
- **Only features that need no database start:** flows, webhooks, idempotent requests, request
  profiling and concurrency tuning (`flow`, `webhook`, `idempotency`, `profiler` and
  `goroutine-engine` in `LYEVE_PLUGINS`). `GET /api/admin/plugins/status` lists the rest as
  needing a database.
- **Replicas do not coordinate on their own.** Without Redis each one fires every cron flow
  unless you turn it off. See [Several replicas](#several-replicas).
:::

## Start it

```bash
docker run -p 3002:3002 \
  -e LYEVE_MODE=stateless \
  -e LYEVE_CONFIG=/etc/lyeve/lyeve.yaml \
  -e RATE_LIMIT_RPS=100 -e SECURE_COOKIE=true \
  -e LYEVE_LICENSE_KEY=... \
  -e STRIPE_WEBHOOK_SECRET=... -e CRM_WEBHOOK_SECRET=... \
  -v ./lyeve.yaml:/etc/lyeve/lyeve.yaml:ro \
  ghcr.io/lyeve-labs/lyeve-core
```

The engine serves one listener, the Content API address (`:3002`). It refuses to start when
`DATABASE_URL` or `DATABASE_REPLICA_URL` is set, or when `MULTI_TENANT` or `LYEVE_SETUP_MODE` is
`true`, because each of them asks for a database. `JWT_SECRET` and `ENCRYPTION_KEY` are not
needed. In production (`APP_ENV` unset or `production`) it still requires `RATE_LIMIT_RPS` and
`SECURE_COOKIE=true`, which turns on HSTS and the HTTPS redirect, and it refuses a `*` in
`CORS_ORIGINS`.

## The configuration file

A `${NAME}` in a value is replaced from the environment, so secrets stay out of the file. The
sections compose through `$include` and `conf.d` like every other setting. See
[Configuration](../reference/configuration.md#the-yaml-file).

```yaml
api_keys:
  - name: operator
    sha256: <hex SHA-256 of the key>
    roles: [super_admin]

flow_variables:
  - key: stripe_secret
    value: ${STRIPE_WEBHOOK_SECRET}
    secret: true

webhooks:
  - name: crm
    url: https://crm.example.com/hooks/lyeve
    secret: ${CRM_WEBHOOK_SECRET}
    max_retries: 0

flows:
  - slug: stripe-relay
    name: Stripe to CRM
    trigger:
      type: trigger.webhook
      config: {secret: "{{ vars.stripe_secret }}"}
    nodes:
      - {id: send, type: webhook.deliver, config: {webhook: crm, event: invoice.paid, payload: "{{ trigger.body }}", wait: true}}
      - {id: check, type: control.condition, config: {expression: "input.status == 'failed'"}}
      - {id: fail, type: control.fail, config: {message: crm unavailable, status: 502}}
      - {id: ok, type: response, config: {status: 202}}
    edges:
      - {from: send, to: check}
      - {from: check, from_port: "true", to: fail}
      - {from: check, from_port: "false", to: ok}
```

### API keys

A key is declared by the hex SHA-256 of its value, never the value. The hash is unsalted, so
generate each key at random and keep the file as private as the keys:

```bash
KEY=$(openssl rand -hex 32)
printf %s "$KEY" | sha256sum
```

A key carries roles or scopes, not both. `admin` and `super_admin` reach the admin routes. A
scope such as `flows:write` limits a key to one resource and action on every route.
`expires_at`, an RFC 3339 time, is optional. A request sends the key in `X-API-Key`.

### Flows

Each entry is a flow definition in the shape the flow import takes, described in the
[flow definition format](../reference/flow-definition.md). Every declared flow is active from
boot. Its id is derived from its slug, so its webhook URL is the same on every replica and after
a restart. List them with `GET /api/admin/flows`.

The engine does not start flows when a definition does not validate, uses a `flow-pro` element
without the license, declares more than twenty flows without it, or declares a slug or a custom
path twice. A webhook trigger checks `X-Flow-Signature`, the hex HMAC-SHA256 of the body with
the secret, with or without a `sha256=` prefix.

### Webhooks

Each entry is an outbound webhook. A failed delivery is retried from memory with increasing
delays, up to `max_retries` (5 when unset). A retry sends the same payload, signed again with a
fresh timestamp.
When its retries are spent the engine publishes `webhook.delivery_failed`. Incoming webhooks are
not served in this mode: a flow's webhook trigger is the inbound side of a relay.

`max_retries`, `retry_delay_seconds`, `payload_template`, `field_filters` and
`jsonpath_filter` need the `webhook-pro` feature. Without it the engine ignores them, logs a
warning, and retries 5 times. Without it the engine also loads only the first 25 declared
webhooks, and logs a warning naming each one it leaves out.

## Durable delivery without a database

Set `max_retries: 0` on a webhook and deliver to it with `wait: true`, as the example does. This
needs the `webhook-pro` feature, because without it `max_retries` is ignored and the webhook
retries 5 times.
Nothing is queued: the flow gets the downstream failure and answers the sender with an error
status, so the sender retries. Stripe, GitHub and Shopify all retry a webhook that fails, which
makes their queue the durable one. A sender that does not retry needs `max_retries` above zero,
and accepts that a retry pending at a restart is lost.

## Several replicas

Webhook and HTTP triggers are served on every replica. Cron flows need one replica to fire
them, and there are two ways to get that.

**With Redis.** Point every replica at the same Redis with `REDIS_URL`. The replicas hold a lease
there, and one at a time fires the cron flows. A replica that shuts down hands the schedule over
on the next tick, within 30 seconds. One that dies hands it over when its lease expires, within
about two minutes. A replica that cannot reach Redis fires nothing rather than risk firing
twice. Replicas compete only with those declaring the same cron flows on the same schedules, so
two relays can share one Redis. The same Redis also shares flow response caches and rate limits
between replicas.

**Without Redis.** Nothing picks a replica, and each fires every cron flow. Set
`FLOW_CRON_ENABLED=false` on all replicas but one:

```bash
docker run ... -e FLOW_CRON_ENABLED=false ghcr.io/lyeve-labs/lyeve-core
```

If that one replica stops, cron stops until you move the setting.

`FLOW_CRON_ENABLED=false` also keeps a replica out of the Redis lease. Unset means on, so a
single replica needs nothing. A value that is not `true` or `false` stops flows rather than
leaving cron on. `GET /api/admin/flows` shows `fires_on_this_replica` on each cron flow.

## Read status

The engine serves no sign-in or setup route in this mode, so admin console v0.22.0 cannot sign
in. Status is read with the engine API, with a key holding `admin` or `super_admin`:

| Route | What it returns |
|---|---|
| `GET /api/admin/mode` | The mode, `database: false`, `metered: false`, the tenant and the declared keys without their hashes |
| `GET /api/admin/plugins/status` | Which features run and which need a database |
| `GET /api/admin/entitlements` | The license in force |
| `GET /api/admin/flows` | The declared flows with their webhook URLs and paths |
| `GET /api/admin/flows/runs` | The latest runs, newest first (`?flow=<slug>`, `?status=`) |
| `GET /api/admin/flows/runs/{run_id}` | One run with its steps |
| `GET /api/admin/webhooks` | The declared webhooks, without secrets, and the retry queue |
| `GET /api/admin/webhooks/deliveries` | The latest deliveries (`?webhook=<name>`) |

`GET /api/admin/plugins/running` also answers a narrower key: one with a scope that covers
`plugins:read`. It lists the running features by name, without reasons or errors.

`GET /readyz` answers `200` once the features have started.

## License

A license works as on any install: a signed token is verified offline, and a license key is
exchanged with the license server and cached in `LYEVE_LICENSE_CACHE_DIR`. Renewing through the
admin API is not available in this mode, so set the key in the environment.

## Related

- [Flows](../features/flows.md): what a flow can do, and the free and `flow-pro` limits.
- [Webhooks](../features/webhooks.md): outbound deliveries and their signatures.
- [Choose where to run LyEve](index.md): the other ways to run the engine.
