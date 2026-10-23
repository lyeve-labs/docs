---
title: Quickstart
description: "Start LyEve on Docker, create the first administrator, get a token, and write and read your first entry in about ten minutes."
sidebar:
  order: 4
---

This is the shortest path from nothing to your first entry. You start a database and the
engine on Docker, then take each step in the admin console or with `curl`. You need Docker,
`curl` and `openssl`.

The API steps call the two APIs the engine serves: the Admin API on port 3001 and the Content
API on port 3002. [Core concepts](../concepts/index.md#the-two-apis) explains the split. The
console steps need the console running beside the engine.
[Run the engine and the console together](engine-and-console.md) starts both with one Compose
file, and its address, `http://localhost:8080`, is where you open the console.

## 0. Start a database and the engine

If you already started an engine by following [Installation](installation.md), skip to
step 1.

```bash
docker network create lyeve

docker run -d --name lyeve-postgres --network lyeve \
  -e POSTGRES_USER=lyeve -e POSTGRES_PASSWORD=lyeve -e POSTGRES_DB=lyeve \
  postgres:16-alpine

until docker exec lyeve-postgres pg_isready -h 127.0.0.1 -U lyeve -q; do sleep 1; done

docker run -d --name lyeve-engine --network lyeve \
  -e APP_ENV=development \
  -e DATABASE_URL="postgres://lyeve:lyeve@lyeve-postgres:5432/lyeve?sslmode=disable" \
  -e JWT_SECRET="$(openssl rand -hex 32)" \
  -e ENCRYPTION_KEY="$(openssl rand -hex 32)" \
  -p 3001:3001 -p 3002:3002 \
  ghcr.io/lyeve-labs/lyeve-core:latest
```

The `until` line waits for PostgreSQL to accept connections, because the engine stops if
the database is not there when it starts. `APP_ENV=development` relaxes the production
checks for a local try, and `sslmode=disable` is fine because both containers share a private
network.

## 1. Check that the engine is up

The first start creates the tables, so give it a few seconds.

<div class="paths">
<div data-path="console">

Open the console. A new install sends you to the setup page, which asks for the setup token
and the first administrator's email address and password. If it says the console cannot reach
the engine, the engine is still starting.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/setup
```

```json
{"setup_required":true,"token_source":"log"}
```

If `curl` cannot connect yet, try again in a few seconds. `setup_required` stays `true` until
the first administrator exists. `token_source` says where the setup token for step 2 is: `log`
when the engine printed one, `env` when you set `LYEVE_SETUP_TOKEN`.

</div>
</div>

## 2. Create the first administrator

Creating the first administrator takes the setup token, so nobody else who can reach the
port claims the install before you do. While no account exists, the engine prints a one-time
token to its log:

```bash
docker logs lyeve-engine 2>&1 | grep setup_token
```

Copy the `setup_token` value from that line. To choose the token yourself, start the engine
with `-e LYEVE_SETUP_TOKEN="$(openssl rand -hex 24)"` instead, 16 characters or more. Do that
whenever more than one engine replica can answer, because each replica prints a token of its
own and setup reaches whichever one the load balancer picks.

The password has to meet the default policy: at least 12 characters, with an upper-case
letter, a lower-case letter and a digit, and not on the list of common passwords.

<div class="paths">
<div data-path="console">

1. On the setup page, paste the token into **Setup token**. With the Compose file from
   [Run the engine and the console together](engine-and-console.md), the token is the
   `SETUP_TOKEN` line of its `.env`.
2. Enter your email address in **Email**, and the password in **Password** and again in
   **Confirm password**.
3. Choose **Create super admin account**.

The console signs you in and opens the **Dashboard**. The account holds the `super_admin` role.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/setup \
  -H "Content-Type: application/json" \
  -d '{
        "email": "you@example.com",
        "password": "Quickstart-Demo-2026",
        "setup_token": "<the token from the log>"
      }'
```

A success answers `201` with the new `user` and a `token`. The account holds the
`super_admin` role. A wrong or missing token answers `401`, and a password that fails the
policy answers `422`. Once the account exists the token stops working, so a second setup
call answers `401`. After a restart with `LYEVE_SETUP_TOKEN` still set, it answers `409`.

</div>
</div>

## 3. Sign in

<div class="paths">
<div data-path="console">

Setup already signed you in. Next time, open the console's address, enter your email address
and password, and choose **Sign in**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{
        "email": "you@example.com",
        "password": "Quickstart-Demo-2026"
      }'
```

The `token` field of the answer is a bearer token for both APIs. Keep it in a variable:

```bash
export TOKEN="<the token from the login answer>"
```

Tokens expire after 15 minutes by default (`JWT_EXPIRY_SECS`). Sign in again when a call
answers `401`. An account with two-factor sign-in gets `"mfa_required": true` and a
`challenge_token` instead of a `token`, and finishes on
[Multi-factor authentication](../features/multi-factor-auth.md).

</div>
</div>

For a script or a CI job, give it a credential of its own rather than your login: an
[API key](../features/api-keys.md) for the Content API or an
[admin token](../how-to/admin-tokens.md) for the Admin API.

## 4. Define a content type

A content type is a name and a list of fields.

<div class="paths">
<div data-path="console">

1. Open **Schema builder** and choose **New schema**.
2. Enter `Post` in **Display name** and `post` in **Machine name**. The machine name is the
   name the APIs use.
3. Choose **Add field**, name it `title`, keep the type `text` and tick **Req**.
4. Choose **Add field** again, name it `body` and pick the type `rich_text`.
5. Choose **Save**. The badge beside the name reads **Saved**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "post",
        "fields": [
          { "name": "title", "field_type": "text", "required": true },
          { "name": "body", "field_type": "rich_text" }
        ]
      }'
```

The answer is `200` with the stored definition, including the `id` field the engine adds.
The key is `field_type`: a
field without it answers `422`, and a misspelled key such as `type` answers `400`.

</div>
</div>

The content type is ready to use as soon as it is saved.
[The data model](../concepts/data-model.md) lists every field type.

## 5. Create an entry

<div class="paths">
<div data-path="console">

1. Open **Content**. With one content type, it opens **Post** straight away.
2. Choose **New entry**.
3. Enter a **Title**, such as `Hello, LyEve`, and some text in **Body**.
4. Choose **Create**. The entry appears in the list.

</div>
<div data-path="api">

Entries are written through the Content API, with the field values under `data`:

```bash
curl -X POST http://localhost:3002/api/v1/content/post \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "data": {
          "title": "Hello, LyEve",
          "body": "First post through the Content API."
        }
      }'
```

The answer is `201` with the entry: its `id`, the values under `data`, and `created_at` and
`updated_at`.

</div>
</div>

## 6. Read it back

<div class="paths">
<div data-path="console">

**Content > Post** lists every entry with its fields. Choose one to open it in the editor. A
site or an app reads the same entries over the Content API, as the other tab shows.

</div>
<div data-path="api">

```bash
curl http://localhost:3002/api/v1/content/post \
  -H "Authorization: Bearer $TOKEN"
```

The answer is a JSON array of the entries of `post`.

</div>
</div>

## Next

- [Your first content type](first-content-type.md): grow `post` into a blog with authors,
  drafts and publishing.
- [Installation](installation.md#run-the-admin-console): run the admin console beside the
  engine.
- [Run the engine and the console together](engine-and-console.md): the same steps in the
  admin console, with a Compose file that starts everything.
- [Developer guide](../guides/developer/index.md): API keys, the client libraries and
  webhooks.
- [API endpoints](../reference/api-endpoints.md): every route on both APIs.
