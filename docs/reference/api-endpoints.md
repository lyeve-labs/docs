---
title: API endpoints reference
description: "Find the route for any job on the Admin API and the Content API, the role it needs, and the page that documents its request and response."
sidebar:
  order: 2
---

This page lists the routes LyEve serves, grouped by what you do with it. Routes that no
feature page covers are listed in full. For a feature's own routes it gives the paths and
links the feature's page, which holds the request and response shapes.

Paths under `/api/admin/` are on the Admin API, port `3001` by default. Paths under `/api/v1/`
are on the Content API, port `3002`. [Core concepts](../concepts/index.md) explains the two.

A route of a feature your license does not cover answers `404` with
`The requested endpoint does not exist.`, the same as a path that was never there. A paid
part of a free feature answers `402` and names the feature it needs, as
[Licensing and tiers](../concepts/licensing-and-tiers.md) shows.

Where one status covers several refusals, the answer carries a `code` beside `error`, such as
`{"error": "a tenant holds at most 500 address entries", "code": "rate_limit.list_full"}`.
Branch on the code, not the message. Each feature page lists its codes.

:::tip[Ask the instance]
`GET /api/admin/openapi/public.json` answers an OpenAPI document of the Content API and every
route outside `/api/admin`, including the URLs your flows answer on. It takes the `admin` or
`super_admin` role. `GET /api/admin/openapi/admin.json` holds the `/api/admin` routes and takes
`super_admin`. Both describe what this instance runs right now, and the admin console shows
them under **API reference**.
:::

## Authenticate

| Credential | Send it as | Use it for |
|---|---|---|
| Session token | `Authorization: Bearer <token>` | A person, or code acting as one. It works on both APIs. The [quickstart](../getting-started/quickstart.md) shows how to get one |
| Session cookie | `__Host-sys_session` when `SECURE_COOKIE=true`, `sys_session` over plain HTTP | A browser, such as the admin console. A request that changes data also sends `X-CSRF-Token` with the `csrf_token` from sign-in |
| API key | `X-API-Key: ly_...` | A site, app or service calling the Content API. The Admin API answers a key that holds the `admin` or `super_admin` role with `401`. See [API keys](../features/api-keys.md) |
| Admin token | `Authorization: Bearer lyat_...` | A script or CI job calling the Admin API. Every other listener answers it with `401`. See [admin tokens](../how-to/admin-tokens.md) |

A super admin picks a tenant with `X-Tenant-ID: <slug>`. A slug that names no tenant answers
`404` with `tenant not found`. The header has no effect for any other caller, who always acts in
the tenant their credential belongs to. See [Tenants](../features/tenants.md).

| Method | Path | Who | Does |
|---|---|---|---|
| `POST` | `/api/admin/auth/login` | Anyone, rate limited | Sign in with `email`, `password` and an optional `tenant`. Answers `user`, `token`, `csrf_token` and `refresh_token`, and sets the session cookie |
| `POST` | `/api/admin/auth/mfa-verify` | Anyone, rate limited | Finish a sign-in that answered `{"mfa_required": true, "challenge_token": "..."}`. Send `challenge_token` and `code` |
| `POST` | `/api/admin/auth/refresh` | Anyone, rate limited | Trade `refresh_token` for a new `token`, `refresh_token`, `csrf_token` and `expires_in`. A refresh token lasts `REFRESH_TOKEN_TTL_SECS`, 30 days by default. Trading the same `refresh_token` again within ten seconds answers the same new token once, then `409` with `AUTH_TOKEN_ROTATED`, and the session stays. A token traded again after that ends the session with `401` |
| `POST` | `/api/admin/auth/logout` | Signed in | End the session. An optional `refresh_token` in the body is revoked too |
| `GET` | `/api/admin/auth/me` | Signed in | The caller's `id`, `email` and `roles` |
| `GET` | `/api/admin/auth/memberships` | Signed in | The tenants the caller belongs to |
| `POST` | `/api/v1/auth/token` | Anyone, rate limited | Sign in on the Content API with `email`, `password` and an optional `tenant`. Answers `token` and `expires_in`, in seconds. An account with a second factor gets the `mfa_required` challenge instead, finished at `mfa-verify` |

A token lasts `JWT_EXPIRY_SECS`, 900 seconds by default. Five failed sign-ins for one account
within 15 minutes lock it for the rest of the window, and the lock answers exactly like a wrong
password.

## Read and write content

These are the Content API routes your sites and apps call. Every one takes a credential: a
session token, or an API key whose scopes allow the action (`read` for `GET`, `write` for
`POST` and `PUT`, `delete` for `DELETE`). A write also needs one of the roles in the **Who**
column, on the token or on the key. `{schema}` is the content type's name. Field values
travel under `data`.

| Method | Path | Who | Does |
|---|---|---|---|
| `GET` | `/api/v1/schemas` | Any credential | List the content types the caller may read, with their fields. The others are left out |
| `GET` | `/api/v1/schemas/{name}` | A caller who may read the type | One content type. Any other caller gets `403` |
| `GET` | `/api/v1/content/{schema}` | Any credential | List entries. Answers a bare array |
| `GET` | `/api/v1/content/{schema}/cursor` | Any credential | List with a cursor, for walking a large set |
| `GET` | `/api/v1/content/{schema}/stream` | Any credential | Follow the content type's changes as Server-Sent Events. See [Follow changes](#follow-changes) |
| `GET` | `/api/v1/content/{schema}/{id}` | Any credential | One entry. An entry that is not published answers `404` unless the caller may read drafts |
| `POST` | `/api/v1/content/{schema}` | `editor`, `admin`, `super_admin` | Create an entry from `{"data": {...}}` |
| `POST` | `/api/v1/content/{schema}/bulk` | `editor`, `admin`, `super_admin` | Create several entries in one call |
| `PUT` | `/api/v1/content/{schema}/{id}` | `editor`, `admin`, `super_admin` | Update the fields the body carries |
| `DELETE` | `/api/v1/content/{schema}/{id}` | `editor`, `admin`, `super_admin` | Delete an entry |
| `PUT` | `/api/v1/content/{schema}/{id}/publish` | `editor`, `admin`, `super_admin` | Publish an entry |
| `PUT` | `/api/v1/content/{schema}/{id}/unpublish` | `editor`, `admin`, `super_admin` | Take an entry back to draft |
| `GET` | `/api/v1/content/{schema}/{id}/relations/{field}` | Any credential | The entries linked through a relation field, as `{"data": [...], "total": n}` |
| `PUT` | `/api/v1/content/{schema}/{id}/relations/{field}` | `editor`, `admin`, `super_admin` | Replace the linked entries |
| `GET` | `/api/v1/content/{schema}/{id}/revisions` | Any credential | An entry's revisions, kept when [content history](../features/content-history.md) is on for the type. Empty when none are stored |
| `PUT` | `/api/v1/content/{schema}/{id}/revisions/{rev_id}/restore` | `editor`, `admin`, `super_admin` | Restore a revision as a new one |

[Access rules](../features/access-rules.md) can narrow any of these per role and content type.

On a content type with draft and publish, a caller may read drafts when it holds both the read
and the update permission on that type. An API key with no `admin` or `super_admin` role also
needs the read and the update scope on it, which `content:read` with `content:write`, or
`content.<schema>:read` with `content.<schema>:update`, grant. Every other caller gets
published entries only. A read by id of a draft or an archived entry answers `404`, and
revisions and related entries leave them out. For a caller who may read drafts, the cursor list
holds entries of every status. A response that can carry drafts is served with
`Cache-Control: private, no-store`, so no shared cache keeps it.

```bash
curl -X POST http://localhost:3002/api/v1/content/article \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"title": "Hello", "body": "First post"}}'
```

### Query parameters

| Parameter | On | Does |
|---|---|---|
| `limit`, `offset` | The list | Page by offset. `limit` runs from 1 to 200 and defaults to 25 |
| `limit`, `cursor` | The cursor list | Page by cursor. `limit` runs from 1 to 1000 and defaults to 20. A value over 1000 gives 20 |
| `filters[<field>]=<value>` | The list | Keep entries whose field equals the value. On a content type with draft and publish, the list holds published entries unless you name `filters[_status]`. A status other than `published` answers `403` unless the caller may read drafts |
| `populate`, `depth` | The list and single reads | Return related entries inline. See [the data model](../concepts/data-model.md) |
| `locale` | The list, the cursor list and single reads | Read the entry in a language, such as `fr` or `pt-BR`. Without it, the `Accept-Language` header decides. It takes effect when [localization](../features/localization.md) runs, which it does on every install unless `LYEVE_PLUGINS` leaves it out |

Use `curl -g` when a URL carries `filters[...]`, so curl does not read the brackets itself.

The cursor list answers `{"data": [...], "next_cursor": "..."}`. Send `next_cursor` back as
`cursor` for the next page. It is empty on the last page, and a page that comes back full
always carries one, so a set whose size is a multiple of `limit` ends on an empty page. Cursor
paging stays fast at depth and never repeats an entry when rows change during the walk. Offset
paging lets you jump to a page number.

### Follow changes

`GET /api/v1/content/{schema}/stream` keeps the connection open and sends one Server-Sent
Event for each entry created, updated or deleted in the content type, in your tenant. It
replays nothing written before it opened.

```bash
curl -N http://localhost:3002/api/v1/content/article/stream \
  -H "Authorization: Bearer $TOKEN"
```

```text
: connected to article stream

data: {"data":{"id":"0f8fad5b-d9cb-469f-a165-70867728950e","title":"Hello"},"event":"after_create","schema":"article"}

: heartbeat
```

- `event` is `after_create`, `after_update` or `after_delete`. `data` is the entry after the
  change, or as it was before a delete, without the fields a read withholds from the caller.
  Relations are not populated and translations are not applied. A change to a draft reaches
  only a caller who may read drafts.
- A `: heartbeat` comment line arrives every 10 seconds, so a proxy keeps an idle stream open.
- When the token expires, the credential is revoked or the caller can no longer read the
  content type, the stream sends one last event named `error`, such as
  `{"error": "credential expired or revoked"}`, and closes.
- A change carries no event `id`, so a client that reconnects cannot resume where it stopped.
  Reconnect with a valid credential, then read the list once to pick up what changed in
  between. A client that reads more slowly than changes arrive misses some of them.
- One credential holds at most 8 open streams and one tenant at most 128. The next one answers
  `429` `too many open change streams` with `Retry-After: 10`.
- A browser's `EventSource` cannot send an `Authorization` header, so read the stream with
  `fetch` or from your server.

### Updating an entry

`PUT /api/v1/content/{schema}/{id}` writes only the fields the body carries. Every other field
keeps its stored value and no default is filled in. A field that is present gets every rule of
the content type, so a `null` or empty value for a required field answers `422` and a unique
value another entry holds answers `409`. A rule that spans fields is checked against the stored
entry with the update laid over it, whenever the body names one of its fields.

An update, or a restored revision, never changes an entry's `_status`. A `_status` in an update
body is left out, so a client that writes back a body it read earlier cannot publish an entry an
editor has since taken back to draft. Move an entry between draft and published with the
`publish` and `unpublish` routes.

### Serve a content type over some transports only

A content type is served over REST, and over [GraphQL](../features/graphql-api.md) and
[gRPC](../features/grpc-api.md) when those features run. An optional `transports` block on the
content type narrows that. Set it when you create or save the content type with
`POST /api/admin/schemas` or `PUT /api/admin/schemas/{name}`:

```json
{
  "name": "article",
  "fields": [{ "name": "title", "field_type": "text" }],
  "transports": { "rest": "rw", "graphql": "r", "grpc": "off" }
}
```

| Mode | Meaning |
|---|---|
| `rw` | Reads and writes. A transport you leave out of the block gets this |
| `r` | Reads only. A write over REST answers `405` with `schema is read-only over this transport` |
| `w` | Writes only. A read answers `405` |
| `off` | Not served. The Content API answers `404`, GraphQL has no type for it, and the gRPC catalog leaves it out |

Over gRPC, the refused direction answers `FAILED_PRECONDITION`. An unknown transport name or mode
answers `422` when you save, so a typo such as `"graphqll": "off"` cannot leave a content type
served where you meant to stop it.

## Admin API routes

Each group below is folded. Open the one you need.

<details>
<summary>First run</summary>

| Method | Path | Who | Does |
|---|---|---|---|
| `GET` | `/api/admin/setup` | Anyone | `{"setup_required": true}` until the first account exists |
| `POST` | `/api/admin/setup` | The setup token | Create the first `super_admin` from `email`, `password` and `setup_token`. The token can come in the `X-Setup-Token` header instead. Answers `201` with `user`, `token` and `csrf_token` |
| `GET` | `/api/admin/setup/status` | The setup token, in `X-Setup-Token` | In setup mode only: which settings are missing, with the lines to set. See [Installation](../getting-started/installation.md) |

</details>

<details>
<summary>Content types</summary>

[Schema changes](../features/schema-changes.md) covers each of these with its body and answer.

| Method | Path | Who | Does |
|---|---|---|---|
| `GET` | `/api/admin/schemas` | Signed in | List content types |
| `GET` | `/api/admin/schemas/{name}` | Signed in | One content type |
| `GET` | `/api/admin/schemas/{name}/stats` | Signed in | How many entries your tenant holds, and when one last changed |
| `POST` | `/api/admin/schemas` | `admin` | Create or save a content type |
| `PUT` | `/api/admin/schemas/{name}` | `super_admin` | Save a content type |
| `DELETE` | `/api/admin/schemas/{name}` | `admin` | Delete a content type |
| `PUT` | `/api/admin/schemas/{name}/rename` | `admin` | Rename a content type |
| `DELETE` | `/api/admin/schemas/{name}/rename` | `admin` | Abandon a rename that stopped partway, under either name. The content type goes back to its old name |
| `PUT` | `/api/admin/schemas/{name}/fields/{field}/rename` | `admin` | Rename a field to `{"new_name": "..."}` |
| `POST` | `/api/admin/schemas/{name}/preview`, `.../preview-ddl` | `admin` | Show the database statements a save would run, without running them. The body is the content type |
| `GET` | `/api/admin/schemas/{name}/history` | `super_admin` | What each save ran against the table, and what is still queued |
| `POST` | `/api/admin/schemas/{name}/apply-pending` | `super_admin`, signed-in session | Run the queued changes for one content type |
| `GET` | `/api/admin/migrate` | `super_admin` | Every queued change on the instance |
| `POST` | `/api/admin/migrate/apply` | `super_admin`, signed-in session | Run every queued change |
| `POST` | `/api/admin/schemas/migrate` | `super_admin` | Run every queued change |
| `GET` | `/api/admin/schemas/export` | `admin` | Download every content type as a bundle |
| `POST` | `/api/admin/schemas/import` | `super_admin` | Load a bundle. See [Moving content types between projects](content-type-portability.md) |
| `GET`, `POST` | `/api/admin/schemas/presets` | `admin` | List presets, or save one of your own. Saving needs the `schema-pro` feature |
| `POST` | `/api/admin/schemas/presets/{id}` | `admin` | Create the content types a preset holds. See [Schema presets](schema-presets.md) |
| `DELETE` | `/api/admin/schemas/presets/{id}` | `admin` | Delete a preset you saved |
| `GET`, `PUT` | `/api/admin/schemas/canvas-layout` | `admin` | The schema canvas layout. Saving it needs the `schema-pro` feature |
| `POST` | `/api/admin/config-sync/export`, `/api/admin/config-sync/diff`, `/api/admin/config-sync/apply` | `super_admin`, signed-in session | Export, compare or apply a configuration bundle. Needs the `config-sync` feature. See [Promote configuration](../features/promote-configuration.md) |

</details>

<details>
<summary>Users, memberships and admin tokens</summary>

| Method | Path | Who | Does |
|---|---|---|---|
| `GET` | `/api/admin/users` | `super_admin` | List users, as a bare array. `limit` runs from 1 to 200 and defaults to 25 |
| `POST` | `/api/admin/users` | `super_admin` | Create a user from `email`, `password` and `roles` |
| `PUT` | `/api/admin/users/{id}/roles` | `super_admin` | Replace a user's roles |
| `PUT` | `/api/admin/users/{id}/state` | `super_admin` | Disable or enable a user with `disabled` |
| `PUT` | `/api/admin/users/{id}/password` | `super_admin` | Set a password from `{"password": "..."}`. It must meet the password policy, or the answer is `422`. Every session the user held ends |
| `DELETE` | `/api/admin/users/{id}` | `super_admin` | Delete a user |
| `GET` | `/api/admin/users/{id}/memberships` | `super_admin`, signed-in session | The tenants a user belongs to, and the roles held in each |
| `PUT` | `/api/admin/users/{id}/memberships` | `super_admin`, signed-in session | Grant a tenant with `{"tenant_id": "...", "roles": [...]}`. Granting it again replaces the roles |
| `DELETE` | `/api/admin/users/{id}/memberships/{tenant}` | `super_admin`, signed-in session | Remove a membership |
| `GET` | `/api/admin/tenant-members/{tenant}` | `super_admin`, signed-in session | Every member of one tenant |
| `GET`, `POST` | `/api/admin/admin-tokens` | `admin`, signed-in session | List admin tokens, or create one |
| `GET` | `/api/admin/admin-tokens/grants` | `admin`, signed-in session | The grants a token can carry |
| `POST` | `/api/admin/admin-tokens/{id}/rotate` | `admin`, signed-in session | Issue a replacement token |
| `DELETE` | `/api/admin/admin-tokens/{id}` | `admin`, signed-in session | Revoke a token |
| `GET` | `/api/admin/admin-tokens/{id}/requests` | `admin`, signed-in session | The token's request log |

A signed-in session means an API key or an admin token is refused with `403`. Creating and
rotating an admin token also take your `password`, or an `mfa_code` when you have a second
factor. [Users, roles and permissions](../guides/user/roles-and-permissions.md) explains roles
and memberships.

</details>

<details>
<summary>Sign in a command-line tool</summary>

A tool on a machine without a browser starts a sign-in, shows the person a code, and polls
until the person approves the code in the admin console.

| Method | Path | Who | Does |
|---|---|---|---|
| `POST` | `/api/admin/auth/device` | Anyone, rate limited | Start a sign-in. Answers `device_code`, `user_code`, `verification_uri`, `verification_uri_complete`, `expires_in` and `interval` |
| `POST` | `/api/admin/auth/device/token` | Anyone, rate limited | Poll with `device_code` until the sign-in is approved |
| `GET` | `/api/admin/auth/device/{user_code}` | `admin`, signed-in session, rate limited | Look up a pending sign-in |
| `POST` | `/api/admin/auth/device/{user_code}/approve` | `admin`, signed-in session, rate limited | Approve it with your `password` or an `mfa_code` |
| `POST` | `/api/admin/auth/device/{user_code}/deny` | `admin`, signed-in session, rate limited | Refuse it |

`verification_uri` is the console page at `LYEVE_CONSOLE_URL` followed by `/admin/device`. A
production instance without `LYEVE_CONSOLE_URL` answers the start with `503` and
`Device sign-in is not configured on this install.`

</details>

<details>
<summary>Features, license and configuration</summary>

| Method | Path | Who | Does |
|---|---|---|---|
| `GET` | `/api/admin/plugins/status` | `admin` | Every feature with `compiled`, `entitled`, `requested` and `active`, its `phase`, a `reason` or `last_error` when it is off, a `manifest` with `label`, `description`, `category` and `maturity`, and the `routes` a running feature serves |
| `GET` | `/api/admin/plugins/running` | Signed in | `plugins`, the features serving your tenant, and `withheld`, the ones that run but are withheld from it |
| `GET` | `/api/admin/plugins/{name}/schema` | `admin` | The JSON Schema of a feature's settings |
| `GET` | `/api/admin/plugins/{name}/config` | `admin` | A feature's saved settings |
| `PUT` | `/api/admin/plugins/{name}/config` | `super_admin` | Save a feature's settings. A key the feature's schema does not list answers `422` with `not a setting of plugin <name>: <keys>`, and nothing is saved |
| `POST` | `/api/admin/plugins/{name}/config/reset` | `super_admin` | Put a feature's settings back to the defaults |
| `GET` | `/api/admin/entitlements` | `admin` | What the license grants. See [Licensing and tiers](../concepts/licensing-and-tiers.md) |
| `GET` | `/api/admin/license` | Signed in | `links` for managing the license, and `renew`, whether the renew route accepts a key |
| `POST` | `/api/admin/license/renew` | `super_admin`, signed-in session | Apply a license with no restart, from `{"license_key": "..."}` |
| `GET`, `PUT` | `/api/admin/tenant-features/{tenant}` | `super_admin`, signed-in session | The features withheld from one tenant |
| `GET` | `/api/admin/config` | `super_admin` | Every setting and the layer it came from |
| `PUT` | `/api/admin/config` | `super_admin` | Store settings in the admin layer. A setting the environment or the file pins is refused and named. See [Configuration](configuration.md) |
| `GET` | `/api/admin/security/controls` | `super_admin` | Which security controls are enforcing. See [Security controls](../trust/security-controls.md) |
| `GET` | `/api/admin/openapi/public.json` | `admin` | OpenAPI for the Content API and every route outside `/api/admin` |
| `GET` | `/api/admin/openapi/admin.json` | `super_admin` | OpenAPI for the `/api/admin` routes |
| `GET` | `/api/admin/openapi.json` | `admin` | The whole document for a super admin, the public half for an admin |

A build without license support answers `404` on both license routes and reports
`"license_module": false` from the entitlements route.

</details>

<details>
<summary>Privacy requests</summary>

| Method | Path | Who | Does |
|---|---|---|---|
| `POST` | `/api/admin/gdpr/export` | `super_admin`, rate limited | Export everything the instance holds about one person |
| `POST` | `/api/admin/gdpr/erase` | `super_admin`, rate limited | Erase it |

[Data protection](../trust/data-protection.md) has the bodies and what each one covers.

</details>

<details>
<summary>Health, metrics and debugging</summary>

| Method | Path | Who | Does |
|---|---|---|---|
| `GET` | `/healthz` | Anyone, on both ports | Liveness: the process is up and the database answers a ping. `503` when the ping fails |
| `GET` | `/readyz` | Anyone, on both ports | Readiness: `503` while starting, while shutting down, when the database is unreachable, or when free disk runs low |
| `GET` | `/startup` | Anyone, on both ports | The first start has finished, for platforms with a separate startup probe |
| `GET` | `/.well-known/jwks.json` | Anyone, Content API only | The public keys that verify LyEve tokens. See [Validate tokens with JWKS](../integrations/validate-tokens-jwks.md) |
| `GET` | `/api/admin/health`, `/api/admin/ready` | Signed in | Ping the database: `{"status": "ok"}`, or `503` with `database unreachable`. A prober without a credential gets `401`, so point probes at the three above |
| `GET` | `/api/v1/health`, `/api/v1/ready` | Signed in | The same on the Content API |
| `GET` | `/api/admin/metrics` | The `METRICS_TOKEN` bearer, or `super_admin` when it is unset | Prometheus metrics. See [Metrics export](../features/metrics-export.md) |
| `GET` | `/api/admin/pool/health` | `admin` | Database connection pool statistics |
| `GET` | `/api/admin/debug/latency` | `admin` | The slowest endpoints by p50, p95 and p99 |
| `GET` | `/api/admin/debug/goroutines` | `admin` | How many goroutines run, and which feature started them |
| `GET` | `/api/admin/debug/gc-config` | `admin` | The garbage collector settings and the memory limit |
| `POST` | `/api/admin/debug/gc-config` | `super_admin` | Change the collector percentage with `{"gogc": <n>}`, with no restart |
| `GET` | `/api/admin/debug/pprof/` | `super_admin` | Go's profiler index. `heap`, `goroutine`, `profile`, `allocs`, `block`, `mutex`, `threadcreate`, `trace`, `cmdline` and `symbol` sit under it |

[Request profiling](../features/request-profiling.md) shows how to read these, and adds its
own routes under `/api/admin/debug/profiler/`.

</details>

## Routes by feature

Each feature serves its routes under the paths below. Its page lists every route with its
body, answer and role.

<details>
<summary>Content, media and search</summary>

| Feature | Paths | Needs |
|---|---|---|
| [Content history](../features/content-history.md) | `/api/admin/content/history/settings`, `/api/admin/content/history/usage`, `/api/admin/content/history/prune`, `/api/admin/content/history?schema=`, and activity at `/api/admin/content/activity`, across tenants at `/api/admin/content/activity/all` | Free. Reading versions older than 30 days needs `content-pro` |
| [Content lifecycle](../features/content-lifecycle.md) | `/api/admin/content` and everything under it: revisions, diffs, schedules, relationships, previews | Free, with 30 days of revisions readable. Older revisions, [releases](../features/content-releases.md) and [comments](../features/editorial-comments.md) need `content-pro` |
| [Editorial review](../features/editorial-review.md) | `/api/admin/review/` | Free, with one workflow per tenant. More, and escalation, quorum and conditions on a stage, need `review-pro` |
| [Media](../features/media.md) | `/api/admin/media`, and `/api/v1/media/{id}/{name}` for a public file | Free, up to 1,000 [image transform](../features/image-transforms.md) variants per tenant. More need `media-pro` |
| [Object storage](../features/object-storage.md) | `/api/admin/storage/`, `/api/v1/storage/local/{provider}` | Free |
| [Bulk import](../features/bulk-import.md) | `/api/admin/imports`, and mapping templates at `/api/admin/imports/templates` | Free. Saving a template, and the date, split and lookup transforms, need `content-pro` |
| [Search](../features/search.md) | `/api/admin/search`, `/api/v1/search/{schema}` | `search` |
| [Localization](../features/localization.md) | `/api/admin/localization/locales`, `/api/admin/localization/rich-text-repairs`, `/api/admin/content/{id}/translations`, `/api/admin/translations/`, `/api/v1/localization/resolve` | Free, with two locales per tenant. More need `localization-pro` |
| [Recommendations](../features/recommendations.md) | `/api/admin/recommendations/`, `/api/v1/recommendations/` | `recommendations` |
| [A/B testing](../features/ab-testing.md) | `/api/admin/ab/experiments`, `/api/v1/ab/expose`, `/api/v1/ab/convert` | `ab-testing` |
| [AI](../features/ai.md) | `/api/admin/ai/`, `/api/admin/content/ai-assist`, `/api/admin/content/ai-usage`, and the support desk at `/api/admin/support/` | `ai`, and `support` for the desk |
| [Data export](../features/data-export.md) | `/api/admin/data-export/` | `data-export` |

</details>

<details>
<summary>Sign-in and access</summary>

| Feature | Paths | Needs |
|---|---|---|
| [Access rules](../features/access-rules.md) | `/api/admin/permissions` | Free, with rules for three roles. More need `rbac-pro` |
| [API keys](../features/api-keys.md) | `/api/admin/api-keys` | Free. A [monthly limit](../features/api-key-request-limits.md) per key needs `apikey-pro` |
| [Password reset](../features/password-reset.md) | `/api/admin/auth/password-reset/` | Free |
| [Magic link sign-in](../features/magic-link-sign-in.md) | `/api/admin/auth/magic-link/` | Free |
| [Rate limiting](../features/rate-limiting.md) | `/api/admin/rate-limits`, with a tenant's own limit at `/api/admin/rate-limits/tenant-global`, address lists at `/api/admin/rate-limits/ip-rules`, and refusals at `/api/admin/rate-limits/history` | Free. Custom rules, tenant limits, address entries and reading refusals need `rate-limit-pro` |
| [Captcha](../features/captcha.md) | `/api/admin/captcha/settings`, and `/api/admin/captcha/site-key`, which needs no credential | Free. Saving a tenant's own provider needs `multitenant-customization` |
| [Multi-factor authentication](../features/multi-factor-auth.md) | `/api/admin/mfa/`, and passkey sign-in at `/api/admin/auth/webauthn/` | Free. Enrolling a passkey needs `mfa-pro` |
| [OAuth sign-in](../features/oauth-sign-in.md) | `/api/admin/auth/oauth/`, `/api/admin/auth/oauth-providers`, `/api/admin/oauth-providers`, `/api/admin/oauth-templates`, `/api/admin/account-links` | Free, with one provider. More need `oauth-pro` |
| [SAML single sign-on](../features/saml-sso.md) | `/api/admin/auth/saml/`, `/api/admin/auth/saml-providers`, `/api/admin/saml-providers`, `/api/admin/saml-templates` | `saml` |
| [SCIM provisioning](../features/scim-provisioning.md) | `/api/admin/scim/providers`, and the SCIM 2.0 service at `/api/v1/scim/v2/` | `scim` |
| [Trusted devices](../features/trusted-devices.md) | `/api/admin/devices`, `/api/admin/session-anomaly/` | `device-fingerprint` |
| [Web application firewall](../features/web-application-firewall.md) | `/api/admin/waf/` | `waf` |
| [PII masking](../features/pii-masking.md) | `/api/admin/pii/` | `pii-mask` |

</details>

<details>
<summary>Tenants</summary>

| Feature | Paths | Needs |
|---|---|---|
| [Tenants](../features/tenants.md) | `/api/admin/tenants`, with cold-storage archives under `/api/admin/tenants/{id}/`, `/api/admin/tenant-domains`, `/api/admin/customization` | Free. Creating a tenant needs `multitenant-provisioning`, and saving customization needs `multitenant-customization` |
| [Tenants](../features/tenants.md) | `/api/admin/migrate/export`, `/api/admin/migrate/import`, `/api/admin/migrate/sqldump/`, `/api/admin/migrate/validate`, `/api/admin/migrate/schemas` | Free |
| [Back up and restore tenants](../how-to/backup-and-restore.md) | `/api/admin/tenant-backup/`, `/api/admin/tenant-clone/` | `tenant-backup` |
| [Migrate existing content](../how-to/migrate-existing-content.md) | `/api/admin/migration/` | `migration-toolkit` |
| [Tenants](../features/tenants.md) | `/api/admin/cost-monitor/` | Free |
| [Usage and quotas](../features/usage-and-quotas.md) | `/api/admin/usage/`, `/api/admin/quotas`, `/api/admin/quota-requests`, `/api/v1/quotas/status`, `/api/v1/quota-requests` | Free. Setting [a tenant's limits](../features/tenant-quotas.md) needs `usage-pro` |
| [Data residency](../features/data-residency.md) | `/api/admin/regions`, `/api/admin/residency/report`, `/api/admin/tenants/{id}/region`, `/api/admin/tenants/{id}/replication`, `/api/admin/tenants/{id}/migrate`, `/api/admin/tenants/{id}/migrations`, `/api/v1/regions` | `data-residency` |

</details>

<details>
<summary>Automation and delivery</summary>

| Feature | Paths | Needs |
|---|---|---|
| [Flows](../features/flows.md) | `/api/admin/flows`, and the flows you publish at `/api/v1/flows/` | Free up to its limit. More needs `flow-pro` |
| [Scheduled jobs](../features/scheduled-jobs.md) | `/api/admin/cron/jobs` (`admin`), `/api/admin/jobs` (`super_admin`) | Free, with email alerts. Slack, Discord, PagerDuty and webhook [alerts](../features/alerts.md) need `alerts-pro` |
| [Idempotent requests](../features/idempotent-requests.md) | `/api/admin/idempotency` | Free |
| [Webhooks](../features/webhooks.md) | `/api/admin/webhooks`, `/api/admin/webhook-dead-letters`, `/api/admin/incoming-webhooks`, and incoming calls at `/api/v1/webhooks/in/{id}` | Free, up to 25 webhooks per tenant. More, and the paid options, need `webhook-pro` |
| [Email](../features/email.md) | `/api/admin/email/`, `/api/admin/email-templates`, and provider callbacks and tracking under `/api/v1/email/` | Free, with one SMTP provider per tenant. More, and the paid routes, need `email-pro` |
| [Event replay](../features/event-replay.md) | `/api/admin/events` | `events` |
| [Message broker events](../features/message-broker-events.md) | `/api/admin/messagebroker/status` | `messagebroker` |
| [Realtime](../features/realtime.md) | `/api/v1/ws/connect`, `/api/v1/ws/broadcast`, `/api/v1/realtime/`, `/api/admin/realtime/metrics`, `/api/admin/ws/metrics` | `realtime` |
| [GraphQL API](../features/graphql-api.md) | `/api/v1/graphql`, `/api/v1/graphql/ws`, `/api/admin/graphql/persisted-queries` | `graphql` |
| [gRPC API](../features/grpc-api.md) | Its own listener, plus `/api/admin/grpc/status` and `/api/admin/grpc/invoke` | `grpc` |

</details>

<details>
<summary>Operations and observability</summary>

| Feature | Paths | Needs |
|---|---|---|
| [Logs](../features/logs.md) | `/api/admin/logs/`, `/api/admin/logging/`, with volume alert rules at `/api/admin/logging/alerts` and one rule at `/api/admin/logging/alerts/{id}` | Free. Paid [alert](../features/alerts.md) channels on a rule need `alerts-pro` |
| [Metrics export](../features/metrics-export.md) | `/api/admin/telemetry/`, with a tenant's own destination at `/api/admin/telemetry/destination` | Free. Saving a tenant destination needs `multitenant-customization` |
| [Error tracking](../features/error-tracking.md) | `/api/admin/error-tracking/`, with triage at `/api/admin/error-tracking/alerts/{id}/resolve`, `/ignore`, `/reopen` and `/assignee`, and spike alerts at `/api/admin/error-tracking/alert-settings` | Free, with 30 days of events. Paid alert channels, a tenant's own threshold and older events need `alerts-pro` |
| [Analytics](../features/analytics.md) | `/api/admin/analytics/`, with the delivery queue at `/api/admin/analytics/deliveries` and `/api/admin/analytics/deliveries/{id}/replay`, `/api/v1/analytics`, and a person's own analytics at `/api/v1/gdpr/export` and `/api/v1/gdpr/erase` | Free. Retry policies, replay and the PostHog, Amplitude and webhook destinations need `analytics-pro` |
| [Request capture](../features/request-capture.md) | `/api/admin/request-captures`, with rules at `/api/admin/request-captures/rules` and replay sets at `/api/admin/request-captures/sets` | Free, kept 24 hours. Creating rules and sets needs `request-capture-pro` |
| [Request profiling](../features/request-profiling.md) | `/api/admin/debug/profiler/` | Free |
| [Concurrency tuning](../features/concurrency-tuning.md) | `/api/admin/debug/goroutines/status`, `/pool`, `/parallel`, `/async-hooks`, `/tuning` | Free to read. Writes to `/pool`, `/parallel` and `/async-hooks` need `goroutine-engine-pro` |
| [Caching](../features/caching.md) | `/api/admin/cache/`, with response cache rules at `/api/admin/cache/rules` | Free, with one provider and 10 rules per tenant. `/api/admin/cache/migrate`, `/api/admin/cache/benchmark` and more rules need `cache-pro` |
| [Audit log](../features/audit-log.md) | `/api/admin/audit-log`, `/api/admin/legal-holds`, `/api/admin/retention-policies`, `/api/admin/retention/` | `audit`. Creating or changing a [stream](../features/audit-log-streaming.md) at `/api/admin/audit-log/sinks` needs `audit-pro` |
| [API analytics](../features/api-analytics.md) | `/api/admin/apianalytics/` | `apianalytics` |
| [Slow query analysis](../features/slow-query-analysis.md) | `/api/admin/query-log` | `query-monitor` |
| [Synthetic monitoring](../features/synthetic-monitoring.md) | `/api/admin/synthetic-monitoring/`, with each probe's notice channels at `/api/admin/synthetic-monitoring/probes/{id}/alert-channels` | Free, with three probes per tenant. More, or intervals under 300 seconds, need `synthetic-monitoring-pro`, and paid notice channels need `alerts-pro` |
| [Replication](../features/replication.md) | `/api/admin/cluster/instances` | `cluster` |

</details>

## Paging on admin lists

| Route | Answers | Paged by |
|---|---|---|
| `GET /api/admin/users` | A bare array | `limit` 1 to 200, default 25, and `offset` |
| API keys, tenants, webhooks, media | `total_count`, `limit`, `offset` | `limit` up to 500, and `offset` |
| Flows and flow runs | `total_count`, `limit`, `offset` | `limit` up to 200, default 50, and `offset` |

## Related

- [Developer guide](../guides/developer/index.md): the order to learn these routes in.
- [Troubleshooting](../guides/developer/troubleshooting.md): what an error answer means.
- [SDK overview](../sdk/index.md): typed TypeScript functions for many of these routes.
