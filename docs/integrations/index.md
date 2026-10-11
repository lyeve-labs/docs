---
title: Connect LyEve to other systems
description: "Pick the right way to connect LyEve to your apps and services: push changes out, read and write content, sign in, store files, send email."
sidebar:
  order: 1
---

Start from what you want to happen, then follow the link. Each page says what
the feature is best for, how to try it and what it needs. A feature that needs
a license starts when your license includes it. See
[Licensing and tiers](../concepts/licensing-and-tiers.md).

## Choose by job

| I want to... | Use | Best for | Free or license |
|---|---|---|---|
| Call my server when content changes | [Webhooks](../features/webhooks.md) | One or a few HTTP endpoints, such as a build hook or a sync job, with signing and a delivery history | Free up to 25 webhooks per tenant. More, templates, filters and retry policies need `webhook-pro` |
| Stream every change to many services | [Message broker events](../features/message-broker-events.md) | Search indexers, caches and pipelines reading NATS, Kafka or RabbitMQ at their own pace | License: `messagebroker` |
| Send a missed window of changes again | [Event replay](../features/event-replay.md) | Catching up a receiver after an outage, or backfilling a new integration | License: `events` |
| Run logic, post to chat or call another API when something happens | [Flows](../features/flows.md) | Several steps, conditions, custom endpoints and outbound calls without writing a service | Free up to 20 flows. Outbound nodes and more flows need `flow-pro` |
| Read and write content from an app or a script | [Content API](../reference/api-endpoints.md#read-and-write-content) | Any language that speaks HTTP and JSON | Free |
| Call LyEve from TypeScript or a frontend framework | [Client libraries](../sdk/index.md) | Typed calls with retries, paging and errors handled, for React, Vue and Svelte too | Free |
| Call LyEve from a server with no user signed in | [API keys](../features/api-keys.md) | Scoped, revocable credentials for backends and jobs | Free. [Monthly request limits](../features/api-key-request-limits.md) per key need `apikey-pro` |
| Fetch exactly the fields a frontend needs, in one request | [GraphQL API](../features/graphql-api.md) | Pages that combine several content types, typed clients, locked-down persisted queries | License: `graphql` |
| Call LyEve from services on my network over HTTP/2 | [gRPC API](../features/grpc-api.md) | Generated clients and frequent calls between services | License: `grpc` |
| Update a browser or app live | [Realtime](../features/realtime.md) | Live previews, dashboards, presence and broadcasts over SSE or WebSocket | License: `realtime` |
| Take payloads from another service in as entries | [Incoming webhooks](../features/webhooks.md#take-in-webhooks-from-other-services) | Shops and form services that post signed JSON | Creating one needs `webhook-pro` |
| Let people sign in through an identity provider | [Sign-in and provisioning](oauth-sso.md) | Google, GitHub or any OIDC provider, SAML 2.0, and SCIM user provisioning | OAuth free with one provider, more need `oauth-pro`. License: `saml` or `scim` |
| Verify LyEve tokens in my own services | [Validate tokens with JWKS](validate-tokens-jwks.md) | Checking a user's token in another service without sharing a secret | Free |
| Keep files in S3 or another object store | [Object storage](../features/object-storage.md) | Several replicas, durable files, uploads straight from the browser | Free |
| Send account mail and my own templated email | [Email](../features/email.md) | An SMTP relay or a pool of providers, branded templates, bounce and click tracking | Free with one SMTP provider and five templates per tenant. A pool, the API transport and more than five templates need `email-pro` |
| Send the audit log to my SIEM | [Audit log streaming](../features/audit-log-streaming.md) | Splunk, Datadog or an HTTPS collector, in order, with retries and a signature | License: `audit` and `audit-pro` |
| Hear in Slack, Discord, PagerDuty or email when a job, an error rate, the log volume or a probe goes wrong | [Alerts](../features/alerts.md) | One message per incident, and one when it recovers | Free by email. Slack, Discord, PagerDuty and webhooks need `alerts-pro` |
| Send analytics events to PostHog, Amplitude or my own endpoint | [Analytics](../features/analytics.md) | Content and product events, with a delivery log, retries and a signed webhook | GA4, Mixpanel, Plausible and Segment free. The rest, and retries, need `analytics-pro` |
| Copy configuration from staging to production | [Promote configuration](../features/promote-configuration.md) | Content types, flows, access rules, webhooks and settings in one bundle, with a dry run | License: `config-sync` |

## Pushing changes out

Four features react to the same content changes, and they suit different
receivers:

| The receiver is | Use |
|---|---|
| An HTTP endpoint you run | [Webhooks](../features/webhooks.md) |
| A message broker with several consumers | [Message broker events](../features/message-broker-events.md) |
| A browser or app with a user in front of it | [Realtime](../features/realtime.md) |
| Any of the above, for changes that already happened | [Event replay](../features/event-replay.md) |

When the reaction needs logic, a [flow](../features/flows.md) starts on the
same events and can call any of them, or an outside API, as one of its steps.

## Reading and writing content

The [Content API](../reference/api-endpoints.md#read-and-write-content) on port
`3002` reads and writes entries, and the Admin API on port `3001` manages
content types, users and settings. [The two APIs](../concepts/index.md) explains
which is which. GraphQL and gRPC serve the same content over other protocols,
and each content type decides which of them may reach it. See
[Transports](../reference/api-endpoints.md#serve-a-content-type-over-some-transports-only).

Every surface keeps tenants apart: a request reaches only its own tenant's
content, and outbound events carry the tenant they came from. See
[Tenants](../features/tenants.md).
