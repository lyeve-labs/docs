---
title: "29. The CLI and the SDK"
description: "Use the commands the server binary answers from a shell, then read trips from TypeScript with the client packages, or from any other language over HTTP."
sidebar:
  order: 29
covers:
  console: []
  api:
    - "GET /api/v1/content/{schema}"
verified:
  engine: v0.62.0
  console: v0.28.0
---

> **Included free** on every install. The GraphQL, gRPC and realtime clients call features that need a license.

There is no separate command-line client to install. The engine's own binary answers a few
commands from a shell, and everything else goes over the two HTTP APIs, which the TypeScript
packages wrap. You need the engine from [lesson 28](backup-and-upgrade.md) and a shell where
it runs.

## 1. Ask the binary for its version and health

The engine image runs one binary, `lyeve`. Reach it in the `engine` service of the stack from
[lesson 1](run-lyeve.md), from the folder that holds its `docker-compose.yml`:

```bash
docker compose exec engine /app/lyeve --version
docker compose exec engine /app/lyeve healthcheck
```

The first prints the version, the commit and the build date. The second asks the engine's own
health probe and exits `0` when it answers `200`. The image uses it as its container health
check. [Server binary](../reference/cli.md) explains both.

## 2. Repair a lost migration record, only when asked

If a start stops with an error that says the schema already holds what a migration creates,
the error names a `lyeve migrate mark-applied` command. Check the database first, then run that
command exactly as the error gives it, from the engine's host or container.
[Record a migration the database already holds](../reference/cli.md#record-a-migration-the-database-already-holds)
says when it refuses. There is no other migration command, because migrations run on every
start.

## 3. Choose a credential for your code

Code that only reads content uses an API key, which lesson 10 created. Code that changes the
content model or settings uses an admin token. Keep the key out of your source, for example in
`LYEVE_API_KEY`, and check it reads trips:

```bash
curl http://localhost:3002/api/v1/content/trip \
  -H "X-API-Key: $LYEVE_API_KEY"
```

## 4. Read trips from TypeScript

1. Pick the packages from the table in [SDK overview](../sdk/index.md#packages). Start with
   `@lyeve-labs/client` and `@lyeve-labs/client-rest`, and add the one for React, Vue or
   Svelte if your app uses it.
2. Install them with `pnpm add`.
3. Copy the example in [Point the client at your instance](../sdk/index.md#point-the-client-at-your-instance),
   put `trip` where it lists `post`, and send the `X-API-Key` header from step 3 in place of the
   token.

Each function maps to one route, so the call in step 3 and `listContent` return the same
trips. An error answer throws an `ApiError` with the status and the parsed body.

## 5. Call the engine from any other language

There is no client package outside TypeScript. Call the routes in
[API endpoints](../reference/api-endpoints.md) directly, with the same header. To check a
LyEve token in a service of your own without sharing a secret, follow
[Validate tokens with JWKS](../integrations/validate-tokens-jwks.md).
[Troubleshooting](../guides/developer/troubleshooting.md) says what each status means.

## Next

[30. Run it in production](production.md) is the last lesson. For depth, read the
[Developer guide](../guides/developer/index.md) and the [SDK overview](../sdk/index.md).
