---
title: Troubleshooting
description: "Find out why a request failed: match the error you got to its cause and fix, then follow its request id into the engine's log."
sidebar:
  order: 5
---

Start with the table: find the status and message you got. If the message is too vague,
follow the request id into the engine's log, which names the cause.

## Errors people hit

| What you see | Cause | Fix |
|---|---|---|
| `401` `authentication required` | No credential, or the token expired. A token lasts 15 minutes by default | Sign in again, or trade your `refresh_token` at `POST /api/admin/auth/refresh` |
| `401` `session invalidated` | The account's password was set again, its sessions were ended, or it was deleted | Sign in again |
| `401` `Invalid email or password.` with the right password | Five failed sign-ins within 15 minutes locked the account. A lock answers exactly like a wrong password, and the log says `login refused: account is locked out` | Wait for the 15 minutes to pass |
| `401` `API keys with an admin role are not accepted on the admin API. Use an admin token.` | An API key holding `admin` or `super_admin` called the Admin API | Use an [admin token](../../how-to/admin-tokens.md) |
| `401` `Admin tokens are accepted on the admin API only.` | An admin token called the Content API or a public route | Use an [API key](../../features/api-keys.md) or a session token there |
| `401` `invalid metrics token` | `METRICS_TOKEN` is set, and the bearer token sent to `/api/admin/metrics` is not it. This includes a super admin's own session token | Send `METRICS_TOKEN` |
| `401` on `POST /api/admin/setup` | The setup token is wrong, or setup already finished and the token was retired | Use the token from the engine's log or `LYEVE_SETUP_TOKEN`. If an account exists, sign in instead |
| `409` `The initial setup has already been completed.` | The first administrator already exists | Sign in |
| `403` `insufficient scope` | The API key's scopes do not cover this method and path | Give the key the scope, such as `content:write`. See [API keys](../../features/api-keys.md) |
| `403` `permission denied` | No access rule lets the caller's roles do this on this content type. A fresh install has no rules, and an API key with no role is refused every content type | Write a rule for the role under **Access > Permissions** and give the key that role. See [access rules](../../features/access-rules.md) |
| `403` `insufficient permissions` | Your role cannot call this route | Ask a `super_admin` for the role. See [roles](../user/roles-and-permissions.md) |
| `403` `request carries no tenant scope` | The account or key belongs to no tenant | Give the account a tenant membership |
| `403` `This route needs a signed-in session.` | An admin token or an API key called a route only a signed-in person may call | Call it from a session |
| `403` `This token is not granted <grant>.` | The admin token's grants do not open this route | Create a token with that grant |
| `404` `The requested endpoint does not exist.` | A wrong path or port, or a feature that is not running | Check the port (`3001` for `/api/admin`, `3002` for `/api/v1`), then `GET /api/admin/plugins/status` |
| `404` `tenant not found` | `X-Tenant-ID` names a slug that does not exist | Check the slug under **Tenants** |
| `402` `payment_required` | A paid part of a feature. `feature` in the body names it | See [Licensing and tiers](../../concepts/licensing-and-tiers.md) |
| `402` `cap_exceeded` | A free tier limit. `cap`, `limit` and `current` say which | A license lifts it |
| `405` `schema is read-only over this transport` | The content type's `transports` block does not allow this direction | See [transports](../../reference/api-endpoints.md#serve-a-content-type-over-some-transports-only) |
| `409` | A slug or unique value is already taken | Pick another value |
| `422` | A required field is missing, a value has the wrong type, or a rule failed | The body names the field |
| `429` `rate limit exceeded` | Too many requests from your address | Wait for the seconds in `Retry-After` |
| `503` with the code `SETUP_REQUIRED` | The engine runs in setup mode | Finish setup. See [Installation](../../getting-started/installation.md) |
| `503` `another instance is applying DDL for this schema; retry in 1s` | Another replica is changing the same content type | Retry after the second in `Retry-After` |
| `503` `{"status":"starting", ...}` | The engine has not finished starting | Retry after the seconds in `Retry-After` |
| `503` with the code `DATABASE_ERROR` | The database is unreachable | Check the database and `GET /api/admin/pool/health` |
| An entry is missing from a list | It belongs to another tenant, it is a draft, or it was soft deleted | A list holds published entries unless you name `filters[_status]`. Check `tenant_id` in the log line |
| The engine refuses to start | A required setting is missing or a production check failed | The start error names every problem. See [Installation](../../getting-started/installation.md) |

## Read the error body

Errors answer JSON. `error` is written for a person, and most answers also carry a `code` to
branch on in code:

```json
{"error": "The requested endpoint does not exist.", "code": "not_found", "request_id": "0af7651916cd43dd8448eb211c80319c"}
```

The message never carries database or driver text. When it is too vague, the log line with the
same request id has the cause.

## Follow a request id

Every response carries an `X-Request-ID` header. The engine takes it from the `X-Request-ID`
you sent, when that is printable ASCII of 128 characters or less. Otherwise it uses the trace
id of a W3C `traceparent` header, or makes a new one. Send your own to trace one call:

```bash
curl -i -H "X-Request-ID: debug-run-42" \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:3002/api/v1/content/article
```

The response carries `X-Request-ID: debug-run-42`, and every log line the request wrote carries
`"request_id": "debug-run-42"`. The [quickstart](../../getting-started/quickstart.md) shows how
to get `TOKEN`.

## Read the engine's log

The engine writes JSON lines to standard output. A line from a request carries `level`
(`DEBUG`, `INFO`, `WARN` or `ERROR`), `msg`, `request_id`, `tenant_id`, `plugin` (the feature
that wrote it) and, with tracing on, `trace_id` and `span_id`. The default level is `INFO`.

A few start-up lines are plain text, so have `jq` skip what does not parse:

```bash
# warnings and errors as they happen
docker logs -f lyeve-engine 2>&1 | jq -cR 'fromjson? | select(.level == "WARN" or .level == "ERROR")'

# everything one request wrote
docker logs lyeve-engine 2>&1 | jq -cR 'fromjson? | select(.request_id == "debug-run-42")'

# everything one tenant did
docker logs lyeve-engine 2>&1 | jq -cR 'fromjson? | select(.tenant_id == "acme")'
```

[Logs](../../features/logs.md), free on every install, adds search, live tail, export and log
levels you change without a restart.

## Where to look next

| Symptom | Look at |
|---|---|
| A feature is not there | `GET /api/admin/plugins/status`. A feature that is off carries a `reason`, such as `not granted by the capability set` |
| A paid feature stopped | `GET /api/admin/entitlements`: `state` and `license_error` |
| A route is slow | `GET /api/admin/debug/latency`, then [Request profiling](../../features/request-profiling.md) |
| Memory or goroutines keep growing | `GET /api/admin/debug/goroutines`, then the heap and goroutine profiles in [Request profiling](../../features/request-profiling.md) |
| Database calls time out | `GET /api/admin/pool/health` |
| The instance is up but sends no traffic | `/readyz`, as [API endpoints](../../reference/api-endpoints.md#admin-api-routes) lists under health |

## Related

- [API endpoints](../../reference/api-endpoints.md): every route and the role it needs.
- [Operator guide](../operator/index.md): watching a running instance.
