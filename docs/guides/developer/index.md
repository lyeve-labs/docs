---
title: Developer guide
description: "Find the shortest path to calling LyEve from your site, app or service, and the page that holds each integration job."
sidebar:
  order: 1
---

This guide is for developers who call LyEve from their own code: a website, a mobile app, a
backend job or another service. Everything goes over HTTP and JSON, so any language works.
The pages it links hold the detail. This page tells you which one to read next.

## Your first hour

1. [Quickstart](../../getting-started/quickstart.md): start an engine, get a token, and write
   and read your first entry with `curl`.
2. [Core concepts](../../concepts/index.md): the two APIs and their ports, tenants and roles,
   in the words the routes use.
3. [Your first content type](../../getting-started/first-content-type.md): model a content type
   with a relation, then read it back filtered and with the related entry inline.
4. [API keys](../../features/api-keys.md): give your site or service its own key, limited to
   the actions it needs.
5. [Read and write content](../../reference/api-endpoints.md#read-and-write-content): every
   Content API route, its query parameters and how paging works.
6. [SDK overview](../../sdk/index.md): the TypeScript client, if you work in TypeScript or
   JavaScript.

## Common jobs

| I need to... | Read |
|---|---|
| Sign in from code and keep the session alive | [Authenticate](../../reference/api-endpoints.md#authenticate) |
| Give a site or service a credential of its own | [API keys](../../features/api-keys.md) |
| Script the Admin API from CI | [Admin tokens](../../how-to/admin-tokens.md) |
| Check a LyEve token inside my own service | [Validate tokens with JWKS](../../integrations/validate-tokens-jwks.md) |
| Page through entries, filter them, include related ones | [Query parameters](../../reference/api-endpoints.md#query-parameters) and [the data model](../../concepts/data-model.md) |
| Publish, unpublish or restore a revision | [Drafts and publishing](../user/drafts-and-publishing.md) |
| Retry a write without creating it twice | [Idempotent requests](../../features/idempotent-requests.md) |
| Shape an endpoint the way my frontend wants it | [Build a custom API with flows](../../how-to/build-a-custom-api-with-flows.md) |
| Query over GraphQL or gRPC | [GraphQL API](../../features/graphql-api.md), [gRPC API](../../features/grpc-api.md) |
| Serve each reader in their language | [Localization](../../features/localization.md) |
| Add full-text search to my site | [Search](../../features/search.md) |
| Keep the content model in my repository | [Schema and flows as code](../../how-to/schema-and-flows-as-code.md) |
| Stay under the rate limits | [Rate limiting](../../features/rate-limiting.md) |
| Find out why a call failed | [Troubleshooting](troubleshooting.md) |

## Hear about changes

| If you want to | Use | Needs |
|---|---|---|
| Run steps of your own when an entry changes, with no server to deploy | A [flow](../../features/flows.md) with an event trigger | Free |
| Have another system called with a signed request when content changes | [Webhooks](../../features/webhooks.md) | Free up to 25 per tenant. More, and retry policies of your own, need `webhook-pro` |
| Push changes to browsers and apps that stay connected | [Realtime](../../features/realtime.md) | `realtime` |
| Feed every change into NATS, Kafka or RabbitMQ | [Message broker events](../../features/message-broker-events.md) | `messagebroker` |
| Send a window of past changes again | [Event replay](../../features/event-replay.md) | `events` |

## Related

- [API endpoints](../../reference/api-endpoints.md): every route on both APIs.
- [Configuration](../../reference/configuration.md): every setting, if you also run the instance.
- [Operator guide](../operator/index.md): deploying what you build against.
