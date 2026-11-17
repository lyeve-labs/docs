---
title: "10. API access and keys"
description: "Choose the right credential for each caller: a token for an app that signs in, an API key for the travel guide's website, an admin token for a deploy script, and the API reference to try any route."
sidebar:
  order: 10
covers:
  console:
    - "/admin/api-access"
    - "/admin/api-keys"
    - "/admin/admin-tokens"
    - "/admin/api-reference"
  api:
    - "POST /api/v1/auth/token"
    - "POST /api/admin/permissions"
    - "GET /api/admin/api-keys"
    - "POST /api/admin/api-keys"
    - "PATCH /api/admin/api-keys/{id}/limits"
    - "POST /api/admin/api-keys/{id}/revoke"
    - "GET /api/admin/admin-tokens/grants"
    - "POST /api/admin/admin-tokens"
    - "GET /api/admin/admin-tokens/{id}/requests"
    - "POST /api/admin/admin-tokens/{id}/rotate"
    - "DELETE /api/admin/admin-tokens/{id}"
    - "GET /api/admin/openapi.json"
verified:
  engine: v0.62.0
  console: v0.28.0
---

So far every API step used your own session token. In this lesson each caller gets the
credential that fits it: the website of the travel guide reads trips with an API key, and a
deploy script calls the Admin API with an admin token. You need the content from
[lesson 4](write-and-publish.md), the super admin from [lesson 1](run-lyeve.md), and a token
in `TOKEN`.

| Caller | Credential | Header |
|---|---|---|
| A person, or an app acting as one | Session token from email and password | `Authorization: Bearer <token>` |
| A website, app or service reading content | API key | `X-API-Key: ly_...` |
| A script or CI job calling the Admin API | Admin token | `Authorization: Bearer lyat_...` |

## Get a token for an app

> **Included free** on every install.

**API access** says how an app authenticates and what each role may do, and hands out a token
to try it.

<div class="paths">
<div data-path="console">

1. Open **Delivery > API access**. **Role capabilities** shows, per role, whether it may **Read
   content**, **Write content** and **Manage content types**.
2. Under **Get an API token**, enter an account's **Email** and **Password** and choose
   **Generate token**.
3. Copy the **Bearer token**. The **Example** above it shows the request to send with it.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3002/api/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "<your password>"}'
```

The answer carries `token` and `expires_in`, in seconds.

</div>
</div>

![API access: the role capabilities table, then an email and password exchanged for a bearer token with Generate token.](./media/api-access-token.mp4)

Give an app that signs in its own account with the `viewer` or `editor` role, never a person's.
Lesson 16 creates accounts.

## Give the website an API key

> **Included free** on every install, up to five active keys per tenant.

A key reads only what its roles may read. The `viewer` role reads nothing until an access rule
opens a content type to it, so open `trip` and `trip_stop` first. Lesson 16 explains access
rules.

<div class="paths">
<div data-path="console">

1. Open **Access > Permissions** and give `viewer` a rule that ticks **Read** on **Trip** and on
   **Trip stop**. [Access rules](../features/access-rules.md#try-it) shows the steps.
2. Open **Access > API keys** and choose **New key**. Enter `travel-site` in **Name** and keep
   **viewer** under **Roles**.
3. Under **Content**, choose **Chosen content types**, then tick read for **Trip** and **Trip
   stop**. Open **Content type catalog** and tick **Discover content types**.
4. Leave **Expires** as it is and choose **Create**.
5. Copy the key from **API key created**. It is not shown again.

</div>
<div data-path="api">

```bash
for type in trip trip_stop; do
  curl -X POST http://localhost:3001/api/admin/permissions \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"role": "viewer", "schema_name": "'"$type"'", "actions": ["read"]}'
done

curl -X POST http://localhost:3001/api/admin/api-keys \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "travel-site", "roles": ["viewer"], "scopes": ["content.trip:read", "content.trip_stop:read", "schemas:read"]}'
```

The answer is `201` and carries the key once, as `raw_key`.

</div>
</div>

Read the published trips with it, the way the website will:

```bash
export KEY="<the key>"

curl http://localhost:3002/api/v1/content/trip \
  -H "X-API-Key: $KEY"
```

![A new API key named travel-site is created with the viewer role and read access to Trip and Trip stop, and the key is shown once to copy.](./media/api-access-new-key.mp4)

The list shows each key's **Roles**, **Scopes**, **Content types**, **Status** and **Usage
(this month)**, and narrows to **Active** or **Revoked** keys. A key with an admin role is
marked **Admin API: refused**, because the Admin API takes admin tokens instead. To retire a
key, revoke it from its row, or `POST /api/admin/api-keys/{id}/revoke` with the `id` from
`GET /api/admin/api-keys`.

## Cap a key's requests

> **Requires** a license with the `apikey-pro` feature.

<div class="paths">
<div data-path="console">

1. On **Access > API keys**, choose **Request limits** on the key's row.
2. Enter the most requests the key may make in an hour, a day and a month. `0` means no limit.
3. Choose **Save**.

</div>
<div data-path="api">

```bash
curl -X PATCH http://localhost:3001/api/admin/api-keys/<key id>/limits \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"hourly_limit": 1000, "daily_limit": 10000, "monthly_limit": 200000}'
```

A key over a limit gets `429` with a `Retry-After` header until its hour, day or month ends,
in UTC.

</div>
</div>

On a free install, saving a limit is refused with a notice that the feature is not enabled on
this engine. With the license, the limits save and show on the key's row.

![Request limits on a free install: Save is refused with a notice that the feature is not enabled on this engine.](./media/api-access-limits-free.mp4)

![Request limits with a license: an hourly, daily and monthly limit are saved and show on the key's row.](./media/api-access-limits-licensed.mp4)

[API keys](../features/api-keys.md) and [API key request limits](../features/api-key-request-limits.md)
cover scopes, writing keys and each key's request history.

## Give a deploy script an admin token

> **Included free** on every install.

An admin token calls the Admin API as you, limited to the grants you choose, one tenant, an
expiry of at most 90 days and the addresses you allow. Every call it makes is logged.

<div class="paths">
<div data-path="console">

1. Open **Access > Admin tokens** and choose **New token**.
2. Enter `deploy-content-types` in **Name**, and tick the grant that reads content types. Open
   **Routes** under a grant to see what it opens.
3. Set **Expires**, and list the addresses the script runs from in **Allowed addresses**.
4. Enter **Your password**, or an **Authentication code** once two-factor is on, and choose
   **Create**.
5. Copy the token from **Admin token**. It is not shown again. **Try it** shows a request that
   uses it.

</div>
<div data-path="api">

`GET /api/admin/admin-tokens/grants` lists the grants. Create the token:

```bash
curl -X POST http://localhost:3001/api/admin/admin-tokens \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "deploy-content-types",
    "grants": ["schemas:read"],
    "expires_at": "2027-01-31T00:00:00Z",
    "password": "<your password>"
  }'
```

The answer carries the token once, as `token`. Set `expires_at` to a date within the next 90
days. On a server, add `"allowed_ips"` with the addresses the script runs from, such as
`["203.0.113.0/24"]`. A call from any other address is refused with `403`, so leave it out while
you try the token on your own computer.

</div>
</div>

The script sends the token like a session token:

```bash
export ADMIN_TOKEN="<the admin token>"

curl http://localhost:3001/api/admin/schemas/export \
  -H "Authorization: Bearer $ADMIN_TOKEN" > travel-guide.yaml
```

![A new admin token named deploy-content-types gets one grant, an expiry and an address range, the password confirms it, and the token is shown once with a request to try it.](./media/api-access-admin-token.mp4)

Each row has its **Request log**, which lists every call, refused ones included, with the
**Time**, **Method**, **Route** and **Address** (`GET /api/admin/admin-tokens/{id}/requests`).
**Rotate** issues a replacement (`POST /api/admin/admin-tokens/{id}/rotate`), and revoking ends
it (`DELETE /api/admin/admin-tokens/{id}`). [Automate the Admin API with admin
tokens](../how-to/admin-tokens.md) walks each of these and what a token may never do.

## Try a route in the API reference

> **Included free** on every install.

The API reference lists every route this instance serves right now, the routes of the features
it runs included.

<div class="paths">
<div data-path="console">

1. Open **Delivery > API reference**. **Public API** lists the Content API and **Admin API** the
   Admin API.
2. Type `content` in **Filter endpoints**, and open `GET /api/v1/content/{schema}`. It shows the
   **Required roles**, the **Parameters**, the **Request body** and the **Response**.
3. Under **Try it**, fill `trip` into the **Path** and choose **Send**. The request goes to this
   engine, signed in as you.
4. **OpenAPI spec** downloads the document for your own tools.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/openapi.json \
  -H "Authorization: Bearer $TOKEN"
```

The answer is the OpenAPI document of every route your role may call.

</div>
</div>

![The API reference is filtered to content routes, the list route for a content type is opened, trip is filled in, and Send shows the answer.](./media/api-access-reference.mp4)

Only a super admin can send a request other than `GET` from the page. For anyone else, the
page shows the complete `curl` line to run instead. Lesson 11 sends requests of every kind from
**API labs**.

## Next

- [11. GraphQL, gRPC, streams and realtime](graphql-and-streams.md): read the trips over other
  transports.
- [API endpoints](../reference/api-endpoints.md): every route, its role and its page.
- [Developer guide](../guides/developer/index.md): the order to learn the APIs in.
