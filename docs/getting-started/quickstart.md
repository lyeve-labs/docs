---
title: Quickstart
description: "Start LyEve on Docker, create the first administrator, get a token, and write and read your first entry in about ten minutes."
sidebar:
  order: 4
---

This is the shortest path from nothing to your first entry. You start a database and the
engine on Docker, then use `curl` for everything else. You need Docker, `curl` and `openssl`.

The steps call the two APIs the engine serves: the Admin API on port 3001 and the Content
API on port 3002. [Core concepts](../concepts/index.md#the-two-apis) explains the split.

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

```bash
curl http://localhost:3001/api/admin/setup
```

```json
{"setup_required":true,"token_source":"log"}
```

The first start creates the tables, so give it a few seconds if `curl` cannot connect yet.
`setup_required` stays `true` until the first administrator exists. `token_source` says where
the setup token for step 2 is: `log` when the engine printed one, `env` when you set
`LYEVE_SETUP_TOKEN`.

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

## 3. Sign in

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

For a script or a CI job, give it a credential of its own rather than your login: an
[API key](../features/api-keys.md) for the Content API or an
[admin token](../how-to/admin-tokens.md) for the Admin API.

## 4. Define a content type

A content type is a name and a list of fields:

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
The content type is ready to use as soon as the call returns. The key is `field_type`: a
field without it answers `422`, and a misspelled key such as `type` answers `400`.
[The data model](../concepts/data-model.md) lists every field type.

## 5. Create an entry

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

## 6. Read it back

```bash
curl http://localhost:3002/api/v1/content/post \
  -H "Authorization: Bearer $TOKEN"
```

The answer is a JSON array of the entries of `post`.

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
