---
title: SDK Overview
description: "Pick the TypeScript client package for calling LyEve, point it at your instance, or call the REST routes from any other language."
sidebar:
  order: 1
---

> **Included free** on every install. The GraphQL, gRPC and realtime clients call features that need a license.

Client packages exist for **TypeScript and JavaScript** only. They are published on npm under
`@lyeve-labs`, and their source is public at
[github.com/lyeve-labs](https://github.com/lyeve-labs). There is no Go, Python or other client.
From any other language, call the HTTP routes directly: each `client-rest` function maps to
one of the routes in [API endpoints](../reference/api-endpoints.md).

:::tip[Best for]
- **A frontend or Node service in TypeScript.** Typed functions for the content and admin
  routes instead of hand-written `fetch` calls.
- **React, Vue or Svelte apps.** Hooks, composables or stores that track loading and errors for
  you.
- **Live updates in the browser.** GraphQL subscriptions or the realtime client, where those
  features are licensed.
:::

## Packages

| Package | What it gives you | Needs |
|---|---|---|
| `@lyeve-labs/client` | The core: `createClient`, `ApiError`, a retrying `fetch` (`createRetryFetch`), a `PaginationIterator` and the shared types. Start here | |
| `@lyeve-labs/client-rest` | One typed function per route for content, content types, users, API keys, permissions, tenants, webhooks and more | |
| `@lyeve-labs/client-react` | React: `CmsProvider`, `useQuery`, `useMutation` | |
| `@lyeve-labs/client-vue` | Vue 3: `CmsPlugin`, `useQuery`, `useMutation` | |
| `@lyeve-labs/client-svelte` | Svelte 5 runes: `createCmsClient`, `createAsyncStore`, `createMutation`, `createAuthStore` | |
| `@lyeve-labs/client-graphql` | `createGraphQLClient`: queries, mutations, and subscriptions over WebSocket | `graphql` |
| `@lyeve-labs/client-grpc` | Schema and content calls over the plain HTTP mirror the gRPC feature serves beside its listener | `grpc` |
| `@lyeve-labs/client-realtime` | `createWSClient` for WebSocket and `SSEClient` for Server-Sent Events | `realtime` |

The last three work only against an instance that runs the feature. See the
[GraphQL API](../features/graphql-api.md), [gRPC API](../features/grpc-api.md) and
[Realtime](../features/realtime.md).

## Point the client at your instance

```bash
pnpm add @lyeve-labs/client @lyeve-labs/client-rest
```

`createClient` takes a `fetch` function and default headers. The `client-rest` functions call
paths such as `/api/admin/schemas` and `/api/v1/content/post`, and the two APIs listen on
different ports, so pass a `fetch` that adds the right origin to each path:

```ts
import { createClient } from "@lyeve-labs/client";
import { getSchemas, listContent } from "@lyeve-labs/client-rest";

const origin = (path: string) =>
  path.startsWith("/api/admin") ? "http://localhost:3001" : "http://localhost:3002";

const lyeveFetch: typeof fetch = (input, init) =>
  fetch(typeof input === "string" && input.startsWith("/") ? origin(input) + input : input, init);

const client = createClient(lyeveFetch, { Authorization: `Bearer ${process.env.TOKEN}` });

const schemas = await getSchemas(client);
const posts = await listContent("post", client, 20);
```

In a browser served from behind a proxy that routes both paths, plain `fetch` works as it is.
For a site that only reads content, send an API key instead of a token:
`{ "X-API-Key": process.env.LYEVE_API_KEY }`. The [quickstart](../getting-started/quickstart.md)
shows how to get a token, and [API keys](../features/api-keys.md) how to make a key.

`createClient` sends `Content-Type: application/json`, gives each request a 15 second timeout
unless you pass your own `signal`, and throws an `ApiError` with `status`, `message` and the
parsed `body` on any error answer. [Troubleshooting](../guides/developer/troubleshooting.md)
says what each status means.

## Related

- [API endpoints](../reference/api-endpoints.md): the routes behind every function.
- [Validate tokens with JWKS](../integrations/validate-tokens-jwks.md): check a LyEve token in
  your own service without a shared secret.
- [Developer guide](../guides/developer/index.md): the order to learn the APIs in.
