---
title: Features
description: "Find the feature for the job you are doing, see whether it is free or needs a license, and where it lives in the admin console."
sidebar:
  order: 1
---

Every feature ships in the same engine image, so there is nothing extra to install. The
tables below group them by the job you are doing. In **Free or license**, **Free** means every
install runs it with no license key, and a name such as `search` means your license has to
carry that feature. Some free features have a paid part, named in the same cell.

**Where in the console** is the path through the admin console's sidebar. An entry shows only
while its feature runs for your tenant, and many show only to a super admin. "None" means the
feature works through the API and settings alone. A route of a feature that is not running
answers `404`. To choose which features start, and to apply a license, see
[Licensing and tiers](../concepts/licensing-and-tiers.md). Prices are at
[lyeve.com/pricing](https://lyeve.com/pricing).

## Model and write content

| Feature | Best for | Free or license | Where in the console |
|---|---|---|---|
| [Schema changes](schema-changes.md) | Creating and changing content types while the engine runs, with a preview first | Free. The schema canvas and presets of your own need `schema-pro`, and moving configuration between instances needs `config-sync` | Schema builder |
| [Content lifecycle](content-lifecycle.md) | Drafts, revisions, rollbacks and scheduled publishing | Free, with 30 days of revision history readable. Older revisions, releases and comment threads need `content-pro` | Content |
| [Content releases](content-releases.md) | Publishing or unpublishing a set of entries together, now or at a set time | `content-pro` | Delivery > Releases, and the Release card on an entry |
| [Editorial comments](editorial-comments.md) | Discussing an entry in threads, with mentions by email | `content-pro` | The Comments panel on an entry |
| [Editorial review](editorial-review.md) | Sign-off in stages before an entry goes live | Free, with one workflow per tenant. More, escalation, quorums and conditional stages need `review-pro` | Reviews |
| [Media](media.md) | A shared file library with thumbnails and public links | Free, up to 1,000 generated image variants per tenant. More need `media-pro` | Media library |
| [Image transforms](image-transforms.md) | Serving an image at any size, crop and format from a signed URL | Free, up to 1,000 stored variants per tenant. More need `media-pro` | Media library, an image's Preview |
| [Bulk import](bulk-import.md) | Loading a CSV, XLSX, JSON, NDJSON or YAML file into a content type | Free. Saved mapping templates and the date, split and lookup transforms need `content-pro` | Insight > Imports |
| [Localization](localization.md) | Translating entries and serving each reader's language | Free, with two locales per tenant. More need `localization-pro` | Settings > Locales, and the Translations panel on an entry |
| [AI](ai.md) | Drafting, translating and generating images with your own AI providers | `ai`. The support desk also needs `support` | Operations > AI |

## Deliver content

| Feature | Best for | Free or license | Where in the console |
|---|---|---|---|
| [Search](search.md) | Full-text and instant search in the admin and on your site | `search` | Search |
| [GraphQL API](graphql-api.md) | Clients that want GraphQL queries and live subscriptions | `graphql` | Delivery > GraphQL |
| [gRPC API](grpc-api.md) | Backend services that speak gRPC | `grpc` | Delivery > gRPC |
| [Realtime](realtime.md) | Pages and apps that update the moment content changes | `realtime` | Insight > Realtime |
| [Recommendations](recommendations.md) | Similar entries, trending entries and a feed for each reader | `recommendations` | Settings > Recommendations |
| [A/B testing](ab-testing.md) | Testing variants against each other on real traffic | `ab-testing` | Delivery > Experiments |

## Automate

| Feature | Best for | Free or license | Where in the console |
|---|---|---|---|
| [Flows](flows.md) | Custom API endpoints and automations, built on a canvas without code | Free, up to twenty flows per tenant on the free triggers and nodes. More needs `flow-pro` | Flows |
| [Scheduled jobs](scheduled-jobs.md) | Calling a URL on a cron schedule, with a history of every run | Free, with failure alerts by email. Slack, Discord, PagerDuty and webhook alerts need `alerts-pro` | Operations > Scheduled jobs |
| [Alerts](alerts.md) | An email, a chat message, a PagerDuty incident or a signed webhook when a job fails, errors spike, log volume jumps or a probe goes down | Free by email. The other channels need `alerts-pro` | Operations > Scheduled jobs, in a job's form. The others over the API |
| [Webhooks](webhooks.md) | Telling another system that content changed, and creating entries from incoming calls | Free, up to 25 webhooks per tenant. More, templates, filters, retry policies and incoming webhooks need `webhook-pro` | Delivery > Webhooks, Incoming webhooks |
| [Message broker events](message-broker-events.md) | Feeding every content change to NATS, Kafka or RabbitMQ | `messagebroker` | Settings > Message broker |
| [Email](email.md) | Transactional mail with templates and delivery tracking | Free, with one SMTP provider and five templates per tenant. More providers, the API transport, more templates and triggers need `email-pro` | Settings > Email |
| [Event replay](event-replay.md) | Re-firing past content events to rebuild an index or recover a downstream | `events` | Insight > Events |

## Sign-in and access

| Feature | Best for | Free or license | Where in the console |
|---|---|---|---|
| [Access rules](access-rules.md) | Deciding what each role may do per content type, flow and review | Free, with rules for up to three roles. More need `rbac-pro` | Access > Permissions |
| [API keys](api-keys.md) | Scripts, apps and CI jobs that read or write content | Free. A monthly request limit per key needs `apikey-pro` | Access > API keys |
| [API key request limits](api-key-request-limits.md) | Capping how many requests one key may make in a month | `apikey-pro` | Access > API keys |
| [Password reset](password-reset.md) | Letting people reset a forgotten password by email | Free | The sign-in page, **Forgot password?** |
| [Magic link sign-in](magic-link-sign-in.md) | Sign-in by a single-use link sent by email | Free | The sign-in page, **Email me a sign-in link** |
| [Multi-factor authentication](multi-factor-auth.md) | A second sign-in step with an authenticator app, backup codes or passkeys | Free. Enrolling a passkey needs `mfa-pro` | Settings > Two-factor authentication |
| [OAuth sign-in](oauth-sign-in.md) | Sign-in with Google, GitHub or any OpenID Connect provider | Free, with one provider. More providers, the Okta and Azure AD templates and role mapping need `oauth-pro` | Access > OAuth providers |
| [SAML single sign-on](saml-sso.md) | Company sign-in through Okta, Azure AD or OneLogin | `saml` | Access > SAML SSO |
| [SCIM provisioning](scim-provisioning.md) | Letting your identity provider create and remove users | `scim` | Access > SCIM provisioning |
| [Trusted devices](trusted-devices.md) | Spotting risky sign-ins and devices a user has not used before | `device-fingerprint` | Settings > Trusted devices |

## Protect the instance

| Feature | Best for | Free or license | Where in the console |
|---|---|---|---|
| [Rate limiting](rate-limiting.md) | Capping how fast clients can call you | Free. Custom rules, sign-in protection changes, tenant limits, address lists and the refusal history need `rate-limit-pro` | Settings > Rate limits |
| [Captcha](captcha.md) | Slowing down password guessing on the admin sign-in | Free. A provider per tenant needs `multitenant-customization` | None. The challenge appears on the sign-in page |
| [Web application firewall](web-application-firewall.md) | Blocking SQL injection, cross-site scripting and similar attacks | `waf` | None |

## Observe and debug

| Feature | Best for | Free or license | Where in the console |
|---|---|---|---|
| [Logs](logs.md) | Searching and tailing what the engine logs, and shipping it to Loki or Elasticsearch | Free, retention policies and volume alerts by email included | Insight > Logs |
| [Error tracking](error-tracking.md) | Grouping repeated errors, triaging them and getting told when they spike | Free, with 30 days of events and spike alerts by email. Chat, PagerDuty and webhook alerts, a tenant's own threshold and older events need `alerts-pro` | Insight > Observability > Errors |
| [Metrics export](metrics-export.md) | Sending metrics to Prometheus, an OTLP collector, Datadog or New Relic | Free, scheduled export included. A destination per tenant needs `multitenant-customization` | Insight > Observability > Telemetry |
| [Request profiling](request-profiling.md) | Finding the slowest endpoints and memory growth | Free | Insight > Observability > Profiler |
| [Request capture](request-capture.md) | Inspecting, comparing and replaying recent requests | Free, replay and diff included, kept 24 hours. Capture rules, longer retention and replay sets need `request-capture-pro` | None |
| [Slow query analysis](slow-query-analysis.md) | Finding requests slowed by the database, with index suggestions | `query-monitor` | Insight > Observability > Slow queries |
| [Synthetic monitoring](synthetic-monitoring.md) | Checking your endpoints on a schedule | Free, with three probes per tenant run at most every 300 seconds. More, or more often, needs `synthetic-monitoring-pro`, and chat, PagerDuty and webhook notices need `alerts-pro` | Insight > Observability > Synthetic monitoring |
| [API analytics](api-analytics.md) | Request volume, errors and latency per endpoint, tenant and client | `apianalytics` | Insight > API analytics |
| [Analytics](analytics.md) | Daily usage figures per tenant, and content events sent to GA4, Mixpanel, Plausible or Segment with a delivery log | Free. Retries, replay and the PostHog, Amplitude and webhook destinations need `analytics-pro` | None |

## Run at scale

| Feature | Best for | Free or license | Where in the console |
|---|---|---|---|
| [Tenants](tenants.md) | Several customers, brands or sites on one instance | Free, with the `default` tenant, domain mapping, the JSON Lines and dump migrate routes, and the cost ledger. More tenants need `multitenant-provisioning`, backup, restore and clone need `tenant-backup`, the database migrator needs `migration-toolkit`, and per-tenant admin branding needs `multitenant-customization` | Operations > Tenants and Tenant costs, Settings > Customization |
| [Replication](replication.md) | Running more than one engine replica against one database | `cluster` | None |
| [Caching](caching.md) | Faster repeated reads, in memory or in Redis, and cached answers for public routes | Free, with one provider of any kind and 10 response cache rules per tenant. Failover, migration and more rules need `cache-pro` | Insight > Observability > Cache |
| [Object storage](object-storage.md) | Keeping files on local disk, S3, MinIO, Google Cloud Storage or Azure Blob | Free, S3, MinIO, Google Cloud Storage, Azure Blob and region pins included | Settings > Storage |
| [Concurrency tuning](concurrency-tuning.md) | Resizing background work on a running instance | Free to read. Changing the pool, parallel engine and async hooks, and their start settings, need `goroutine-engine-pro` | Insight > Observability |
| [Idempotent requests](idempotent-requests.md) | Retrying a write safely, so it runs once | Free, keys shared across instances through Redis included | Insight > Observability > Idempotency |
| [Usage and quotas](usage-and-quotas.md) | Metering traffic per key and tenant, and monthly snapshots for billing | Free. Setting a tenant's limits needs `usage-pro` | Operations > Quotas and usage |
| [Tenant quotas](tenant-quotas.md) | Holding each tenant to request, storage and bandwidth limits, and approving requests for more | `usage-pro` | Operations > Quotas and usage |
| [Promote configuration](promote-configuration.md) | Moving content types, flows, access rules, webhooks and settings to another instance in one bundle | `config-sync` | Settings > Config sync |

## Data and compliance

| Feature | Best for | Free or license | Where in the console |
|---|---|---|---|
| [Audit log](audit-log.md) | A record of who did what in the admin, with retention and legal holds | `audit`. Streaming to an outside sink also needs `audit-pro` | Insight > Audit log |
| [Audit log streaming](audit-log-streaming.md) | Sending the audit log to Splunk, Datadog or your own HTTPS endpoint as it is written | `audit-pro`, on a license that also carries `audit` | Insight > Audit log, Streaming |
| [Data export](data-export.md) | Exporting content to files or S3, on demand or on a schedule | `data-export` | Insight > Data export |
| [PII masking](pii-masking.md) | Hiding personal data in responses and logs, and seeing who read it unmasked | `pii-mask` | Settings > PII masking |
| [Data residency](data-residency.md) | Keeping each tenant's data in its assigned region | `data-residency` | Settings > Data residency |

## Built into every install

These screens come with the engine rather than with a feature, so no feature list turns
them off. Users and the Settings screens below show only to a super admin.

| Screen | What it is for | Read more |
|---|---|---|
| Access > Users | Creating accounts and giving them roles | [Roles and permissions](../guides/user/roles-and-permissions.md) |
| Access > Admin tokens | Credentials for scripts that call the Admin API | [Admin tokens](../how-to/admin-tokens.md) |
| Delivery > API access, API reference, API labs | How apps authenticate, every route the instance serves, and a place to send a real request | [The two APIs](../concepts/index.md#the-two-apis) |
| Operations > Plugins | Which features run on this instance, and their settings | [Licensing and tiers](../concepts/licensing-and-tiers.md) |
| Settings > License | The license the instance runs on, and where a new key is activated | [Licensing and tiers](../concepts/licensing-and-tiers.md) |
| Settings > Configuration | Every setting the engine reads, and where each value came from | [Configuration](../reference/configuration.md) |
| Settings > Security | Which protections are on right now | [Security controls](../trust/security-controls.md) |
| Settings > Privacy requests | Exporting or erasing everything held about one person | [Data protection](../trust/data-protection.md) |

The [User guide](../guides/user/index.md) walks through the screens an editor uses.
