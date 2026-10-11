---
title: Licensing & Tiers
description: "See what LyEve runs for free, what a license adds, how to apply a license key, and what happens when there is none or it expires."
sidebar:
  order: 5
---

LyEve runs the same engine image with or without a license. With no key it runs
the free tier: the core CMS and every free feature, for production use and with
no time limit. A license adds the paid features and capabilities it names. This
page covers what each side holds, how to apply a license, how to choose which
features start, and how to check what is running. Prices and plans are at
[lyeve.com/pricing](https://lyeve.com/pricing).

| If you want to | Read |
|---|---|
| Know whether something costs money | [What runs free](#what-runs-free) and [What a license adds](#what-a-license-adds) |
| Put a license on an install | [Apply a license](#apply-a-license) |
| Know what happens when a license lapses | [Without a license, and when one expires](#without-a-license-and-when-one-expires) |
| Run fewer features | [Choose which features start](#choose-which-features-start) |
| See what is on right now | [Check what is running](#check-what-is-running) |

## What runs free

The core CMS is free: content types and entries, drafts and revisions, users and
roles, sign-in, the Admin API and the Content API, on PostgreSQL, MySQL and SQL
Server.

These features run on every install with no license key. The name is what
`LYEVE_PLUGINS` and `GET /api/admin/entitlements` call the feature. Some are free
up to a ceiling or hold back a paid part, and the capability in the last column
lifts the ceiling or opens the part. Each feature's page names exactly what it
gates.

| Feature | Name | Free | Paid part |
|---|---|---|---|
| [Access rules](../features/access-rules.md) | `permissions` | Rules for three roles | `rbac-pro` |
| [Analytics](../features/analytics.md) | `analytics` | The GA4, Mixpanel, Plausible and Segment destinations, one attempt per event, and the delivery log | `analytics-pro` |
| [API keys](../features/api-keys.md) | `apikey` | Five active keys per tenant, thirty days of each key's request history, no monthly limit per key | `apikey-pro` |
| [Bulk import](../features/bulk-import.md) | `bulk-import` | Every format, XLSX included, and the trim, lowercase and uppercase transforms | `content-pro` |
| [Caching](../features/caching.md) | `cache` | One cache provider of any kind, and 10 response cache rules per tenant | `cache-pro` |
| [Captcha](../features/captcha.md) | `captcha` | One provider for the instance | `multitenant-customization` |
| [Concurrency tuning](../features/concurrency-tuning.md) | `goroutine-engine` | Every view of the worker pool, the parallel engine, async hooks and task status | `goroutine-engine-pro` |
| [Content lifecycle](../features/content-lifecycle.md) | `content` | Everything except releases and comment threads, with every revision kept and the last 30 days of them readable | `content-pro` |
| [Editorial review](../features/editorial-review.md) | `review` | One workflow per tenant, with every stage feature except escalation, quorum and conditions | `review-pro` |
| [Email](../features/email.md) | `email` | One SMTP provider and five templates per tenant | `email-pro` |
| [Error tracking](../features/error-tracking.md) | `error-tracking` | Grouping, triage and spike alerts by email, with 30 days of events readable | `alerts-pro` |
| [Flows](../features/flows.md) | `flow` | Twenty flows per tenant, on the nodes that stay inside the instance | `flow-pro` |
| [Idempotent requests](../features/idempotent-requests.md) | `idempotency` | | |
| [Localization](../features/localization.md) | `localization` | Two locales per tenant | `localization-pro` |
| [Logs](../features/logs.md) | `logging` | Everything, with volume alerts by email | `alerts-pro` |
| [Magic-link sign-in](../features/magic-link-sign-in.md) | `magic-link` | | |
| [Media](../features/media.md) | `media` | 1,000 generated image variants per tenant | `media-pro` |
| [Metrics export](../features/metrics-export.md) | `telemetry` | Every exporter for the instance | `multitenant-customization` |
| [Multi-factor authentication](../features/multi-factor-auth.md) | `mfa` | TOTP, backup codes and signing in with an enrolled passkey | `mfa-pro` |
| [OAuth sign-in](../features/oauth-sign-in.md) | `oauth` | One provider per install, from the Google, GitHub or Apple template or by hand | `oauth-pro` |
| [Object storage](../features/object-storage.md) | `storage` | Everything, region pins included | |
| [Password reset](../features/password-reset.md) | `password-reset` | | |
| [Rate limiting](../features/rate-limiting.md) | `rate-limit` | The global limit and the built-in protections | `rate-limit-pro` |
| [Request capture](../features/request-capture.md) | `request-capture` | Every request kept 24 hours, with replay and diff | `request-capture-pro` |
| [Request profiling](../features/request-profiling.md) | `profiler` | | |
| [Scheduled jobs](../features/scheduled-jobs.md) | `cron` | Every job, with failure alerts by email | `alerts-pro` |
| [Schema changes](../features/schema-changes.md) | `schema` | The list editor, every schema change, the built-in presets and applying any preset | `schema-pro`, `config-sync` |
| [Synthetic monitoring](../features/synthetic-monitoring.md) | `synthetic-monitoring` | Three probes per tenant, each run at most every 300 seconds, with down notices by email | `synthetic-monitoring-pro`, `alerts-pro` |
| [Tenants](../features/tenants.md) | `multitenant` | One tenant, `default` | `multitenant-provisioning` and the other tenant capabilities below |
| [Usage and quotas](../features/usage-and-quotas.md) | `usage` | Metering, and enforcing the quotas already stored | `usage-pro` |
| [Webhooks](../features/webhooks.md) | `webhook` | 25 outbound endpoints per tenant | `webhook-pro` |

A free install also holds up to three admin seats: accounts with the `admin` or
`super_admin` role, in their own tenant or through a tenant membership. Each
account counts once across every tenant, disabled accounts included, and the
first account, created at setup, is one of the three. Editors and viewers never
count. A license that is `active` or in `grace` lifts that
ceiling, whatever names it carries. Creating a fourth admin, promoting an account
to either role, or granting a membership with either role answers
`{"error": "cap_exceeded", "cap": "admin.seats", "limit": 3, "current": 3, "upgrade_url": ""}`.
When a license lapses, every admin keeps their role and keeps signing in.

Every install also holds these capabilities, which are about where a free
feature keeps its data:

| Capability | What it lets you do |
|---|---|
| `cache-redis` | Keep the cache in Redis. |
| `storage-s3` | Store media in S3 or an S3-compatible service. |
| `logging-retention` | Keep logs for as long as your retention policy says. |
| `rate-limit-distributed` | Share rate limit counters across replicas through Redis. |
| `idempotency-distributed` | Share idempotency keys across replicas through Redis. |
| `request-capture-replay` | Replay captured requests. |
| `telemetry-streaming` | Export metrics on a schedule. |
| `cost-monitor` | Track the cost of each tenant. |

Backing services such as Redis or S3 need no license. Configure the service and
the free feature uses it.

Past a ceiling, the request answers `402 Payment Required` with a body that
names it, and nothing you already hold is taken away. Only the next object is
refused:

```json
{"error": "cap_exceeded", "cap": "rbac.roles", "limit": 3, "current": 3, "upgrade_url": ""}
```

## What a license adds

A license carries a list of names. Each paid feature runs when its name is on the
list:

| Feature | Name |
|---|---|
| [A/B testing](../features/ab-testing.md) | `ab-testing` |
| [AI](../features/ai.md) | `ai` |
| [API analytics](../features/api-analytics.md) | `apianalytics` |
| [Audit log](../features/audit-log.md) | `audit` |
| [Data export](../features/data-export.md) | `data-export` |
| [Data residency](../features/data-residency.md) | `data-residency` |
| [Event replay](../features/event-replay.md) | `events` |
| [GraphQL API](../features/graphql-api.md) | `graphql` |
| [gRPC API](../features/grpc-api.md) | `grpc` |
| [Message broker events](../features/message-broker-events.md) | `messagebroker` |
| [PII masking](../features/pii-masking.md) | `pii-mask` |
| [Realtime](../features/realtime.md) | `realtime` |
| [Recommendations](../features/recommendations.md) | `recommendations` |
| [Replication](../features/replication.md) | `cluster` |
| [SAML single sign-on](../features/saml-sso.md) | `saml` |
| [SCIM provisioning](../features/scim-provisioning.md) | `scim` |
| [Search](../features/search.md) | `search` |
| [Slow query analysis](../features/slow-query-analysis.md) | `query-monitor` |
| [Trusted devices](../features/trusted-devices.md) | `device-fingerprint` |
| [Web application firewall](../features/web-application-firewall.md) | `waf` |

Other paid names open part of a feature. The feature runs either way, and the
license opens the part:

| Name | Part of | What it opens |
|---|---|---|
| `flow-pro` | Flows | More than twenty flows, custom endpoint paths, webhooks, other protocols, outbound nodes and datasources. |
| `webhook-pro` | Webhooks | More than 25 endpoints, payload templates and filters, retry policies, dead-letter replay and dismiss, the payload preview, delivery search, retrying one delivery, and creating or editing incoming webhooks. |
| `email-pro` | Email | A second provider and more, the API transport, more than five templates, creating or editing triggers, the delivery dashboard and bounce classification. |
| `localization-pro` | Localization | More than two locales. |
| `oauth-pro` | OAuth sign-in | More than one provider, the Okta and Azure AD templates, role mapping, default roles of your own and a backup provider. |
| `mfa-pro` | Multi-factor authentication | Enrolling a passkey and marking one as the backup key. |
| `synthetic-monitoring-pro` | Synthetic monitoring | More than three probes, and intervals down to 15 seconds. |
| `rate-limit-pro` | Rate limiting | Custom rate limit rules, changes to the sign-in protections, a tenant's own global limit, address allow and deny lists, and the refusal history. |
| `rbac-pro` | Access rules | More than three roles that hold rules. |
| `apikey-pro` | API keys | More than five active keys per tenant, the whole request history, and [monthly request limits](../features/api-key-request-limits.md): setting or raising a key's limit. |
| `usage-pro` | Usage and quotas | [Tenant quotas](../features/tenant-quotas.md): setting a tenant's request, storage and bandwidth limits, and turning hard limits and blocking on. |
| `alerts-pro` | Scheduled jobs, error tracking, logs, synthetic monitoring | [Alerts](../features/alerts.md) in Slack, Discord, PagerDuty or a signed webhook, a tenant's own error spike threshold, and error events older than 30 days. |
| `content-pro` | Content lifecycle, bulk import | [Content releases](../features/content-releases.md), [editorial comments](../features/editorial-comments.md), [revisions older than 30 days](../features/content-lifecycle.md#how-far-back-you-can-read), and saved import mapping templates with the date, split and lookup transforms. |
| `review-pro` | Editorial review | More than one workflow per tenant, escalation on a missed deadline, stages that need several approvals, and stages that apply only to some entries. |
| `request-capture-pro` | Request capture | Capture rules per route with sampling and retention up to 30 days, and saved replay sets. |
| `cache-pro` | Caching | Several cache providers with failover, moving entries between providers, the provider benchmark, and more than 10 response cache rules per tenant. |
| `goroutine-engine-pro` | Concurrency tuning | Changing the worker pool, the parallel engine and async hooks on a running server, and the four settings that set them at start. |
| `analytics-pro` | Analytics | Retry policies, replaying a failed delivery, and the PostHog, Amplitude and webhook destinations. |
| `media-pro` | Media | More than 1,000 stored [image transform](../features/image-transforms.md) variants per tenant. |
| `schema-pro` | Schema changes | [The schema canvas](../features/schema-changes.md#work-on-the-schema-canvas) with a layout every admin of the tenant shares, and [saving presets of your own](../reference/schema-presets.md#save-a-preset-of-your-own). |
| `config-sync` | Schema changes | [Promoting the configuration](../features/promote-configuration.md): content types, flows, access rules, webhooks and settings to another instance as one bundle. Moving one content type or one flow stays free. |
| `multitenant-provisioning` | Tenants | Creating tenants. |
| `tenant-backup` | Tenants | Backing up, restoring and cloning a tenant. |
| `migration-toolkit` | Tenants | Jobs that copy a database to another one, across database types, with cancel and rollback. |
| `multitenant-customization` | Tenants, captcha, metrics export | The admin console branded per tenant, a [captcha provider](../features/captcha.md#give-a-tenant-its-own-captcha) per tenant, and a [metrics destination](../features/metrics-export.md#send-a-tenant-its-own-metrics) per tenant. |
| `audit-retention` | Audit log | Retention policies and legal holds. A license with `audit` includes it. |
| `audit-pro` | Audit log | [Streaming the log](../features/audit-log-streaming.md) to Splunk, Datadog or HTTPS. It needs `audit` on the same license, and a license with `audit` does not include it. |
| `support` | AI | The support desk. It needs `ai` on the same license. |

A license that carries a free feature's own name opens that feature's paid part
too: `flow`, `webhook`, `email`, `localization`, `mfa`, `oauth`,
`synthetic-monitoring` and `rate-limit` open their `-pro` part, and `multitenant`
opens `multitenant-provisioning`, `tenant-backup` and `migration-toolkit`.

These names are in beta: `alerts-pro`, `analytics-pro`, `apikey-pro`,
`audit-pro`, `cache-pro`, `config-sync`, `content-pro`, `flow-pro`,
`goroutine-engine-pro`, `localization-pro`, `media-pro`, `migration-toolkit`,
`multitenant-customization`, `multitenant-provisioning`, `request-capture-pro`,
`review-pro`, `schema-pro`, `support`, `synthetic-monitoring-pro`,
`tenant-backup` and `usage-pro`. So are these paid features: `ai`, `cluster`,
`data-residency`, `device-fingerprint`, `messagebroker`, `query-monitor`,
`recommendations` and `saml`. Each works as its page describes, and its fields
can still change before it is marked stable. Each page says what keeps working
when the license lapses.

Which names each plan carries is on [lyeve.com/pricing](https://lyeve.com/pricing).
A feature can also be bought on its own, and however a name reached the license,
it works the same.

A paid feature the license does not cover does not start, so its routes answer
`404`. One that was running when the license lapsed can answer `402` until the
next restart. A paid part of a running feature answers `402` with the name it
needs:

```json
{"error": "payment_required", "plugin": "multitenant-provisioning", "feature": "feature:multitenant_provisioning", "upgrade_url": ""}
```

## Apply a license

You receive a license as a signed token, or as a license key that starts with
`lyeve_live_` (`lyeve_test_` for a test key). Either works in both places below.

**In the environment.** Set `LYEVE_LICENSE_KEY` and restart the engine:

```bash
docker run -e LYEVE_LICENSE_KEY="<token or key>" ... ghcr.io/lyeve-labs/lyeve-core:latest
```

![Settings > License: the current plan and status, links to plans and to the customer portal, and a box to paste a license token or key.](images/license.png)

**In the admin console.** A super admin opens **Settings > License**, pastes
the token or key into **License token or key**, and selects
**Activate license**, or **Renew license** when one is already in force. The
same call over the API:

```bash
curl -X POST http://localhost:3001/api/admin/license/renew \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"license_key": "<token or key>"}'
```

```json
{"plan": "pro", "state": "active", "features": ["..."], "license_source": "token", "expires_at": "2027-01-31T00:00:00Z"}
```

The route takes the `super_admin` role and a signed-in session, not an admin
token or an API key. The instance stores what you enter, and encrypts a key with
`ENCRYPTION_KEY`, so a restart and every replica pick it up.

What changes at once and what waits for a restart:

- Paid parts of a running feature, such as `flow-pro` or
  `multitenant-provisioning`, apply to the next request.
- A paid feature that was not running at boot starts at the next restart.

| Answer | Meaning |
|---|---|
| `200` | Applied. The body is the new entitlement. |
| `400` `invalid JSON` | The body is not JSON. |
| `400` `license_key is required` | The body carried no key. |
| `400` `license verification failed` | The token does not verify. |
| `400` `the license server refused this key` | The key is invalid, revoked or out of activations. |
| `409` | `LYEVE_LICENSE_KEY` holds a license key, and the environment wins. Change the key there. |
| `502` | The license server could not be reached, or returned a token that does not verify. |
| `503` `license key exchange is not configured on this instance` | `LYEVE_LICENSE_SERVER_URL` is not usable, so a key cannot be exchanged. |

### A token works offline, a key is exchanged

A **signed token** is verified on the instance with no network call, so an
air-gapped install behaves like any other. The check cannot be turned off, and a
development or staging install is checked the same way. See
[Verify offline](../trust/verify-offline.md).

A **license key** is exchanged for a token:

1. At start the engine sends the key and a random id for the install to
   `LYEVE_LICENSE_SERVER_URL`. It accepts `https` only and never follows a
   redirect.
2. It verifies the token that comes back exactly as it verifies one you paste. A
   token that does not verify grants nothing.
3. It caches the token in `LYEVE_LICENSE_CACHE_DIR` with a digest of the key,
   never the key itself. A later start uses the cached token with no network
   call until the token is within seven days of its expiry.
4. From then on it exchanges the key again. A running engine checks every six
   hours, and also retries while it holds no active token.

Nothing else is sent: no content, users, configuration or usage. When the server
refuses the key (`invalid_key`, `key_revoked` or `quota_exceeded`), the cached
token is discarded and the install runs the free tier. When the server cannot be
reached, a cached token keeps working, and with none the install runs the free
tier. Any other answer, such as a proxy's `403`, counts as unreachable. The
reason is logged and reported as `license_error` by `GET /api/admin/entitlements`.

Mount `LYEVE_LICENSE_CACHE_DIR` on a volume so the cached token survives a new
container.

## Without a license, and when one expires

| State | When | Paid features |
|---|---|---|
| `free` | No license, none that verifies, or a license whose grace has ended | Off |
| `active` | The license verifies and has not expired | On |
| `grace` | Expired less than seven days ago | Still on, with a warning in the log every hour |

The engine checks the clock every hour. An active license past its expiry moves
to `grace`, and grace that ends moves the install to `free`, both without a
restart. The log also warns 30, 14, 7, 3 and 1 days before expiry.

In every state the free tier keeps running. A missing or expired license never
takes a free feature away, and a paid feature that stops keeps its data. Past a
free ceiling after a downgrade, everything you hold stays, and only the next
object is refused. Renew before grace ends and the paid features carry on
without a gap.

## Choose which features start

A feature runs when all three hold:

| Condition | Set by | Meaning |
|---|---|---|
| Included | The engine image | The published image includes every feature. |
| Licensed | `LYEVE_LICENSE_KEY` | The feature is free, or the license carries it. |
| Requested | `LYEVE_PLUGINS` | The feature is on your list. Empty means every licensed feature. |

`LYEVE_PLUGINS` is a comma-separated list of feature names, with whitespace
around each name trimmed and names matched exactly. Once set, only the listed
features start, free ones included, plus any licensed feature a listed one
depends on. It can narrow what runs but never widen it, so naming a feature your
license does not carry has no effect. Use it for a lean deployment with fewer
routes:

```bash
LYEVE_PLUGINS=apikey,multitenant,content,media,storage,schema,search
```

| Licensed? | In `LYEVE_PLUGINS`? | Result | Reason reported |
|---|---|---|---|
| Yes | Yes, or the list is empty | Running | none |
| Yes | No | Off | `granted but not requested via LYEVE_PLUGINS` |
| No | Yes | Off | `requested via LYEVE_PLUGINS but not granted by the capability set` |
| No | No | Off | `not granted by the capability set` |

A listed name the image does not include logs
`LYEVE_PLUGINS requested a plugin not compiled into this binary` at start and
reports `plugin not compiled into this build`, so a typo is easy to find.

## Check what is running

**In the admin console**, **Operations > Plugins** lists every feature with its
state and, for one that is off, the reason. Admins and super admins see it.
**Settings > License** shows the plan, its state, the expiry and any license
problem.

**Over the API**, three routes answer:

```bash
curl http://localhost:3001/api/admin/entitlements \
  -H "Authorization: Bearer $TOKEN"
```

`GET /api/admin/entitlements` takes the `admin` or `super_admin` role and reports
the license in force. A free install answers:

```json
{
  "plan": "free",
  "plan_label": "Free",
  "state": "free",
  "features": ["analytics", "apikey", "bulk-import", "cache", "cache-redis", "..."],
  "tenant_quota": 1,
  "withheld": [],
  "caps": {"admin.seats": 3},
  "license_module": true
}
```

<details>
<summary>Entitlement fields</summary>

| Field | Meaning |
|---|---|
| `plan`, `plan_label` | The plan, and the label the admin console shows. |
| `state` | `free`, `active` or `grace`. |
| `features` | Every name the install may run, sorted: the license's names, the free features and the free capabilities. |
| `tenant_quota` | How many tenants the install may hold. `0` means unlimited. |
| `caps` | Each ceiling the license decides. `0` means unlimited. |
| `withheld` | Features a super admin withheld from the caller's tenant. See [Tenants](../features/tenants.md). |
| `license_source` | `token`, `key` or `stored_key`, when a license is set. |
| `expires_at`, `grace_ends_at` | When the license expires, and when grace ends while in grace. |
| `license_error` | Why the last key exchange failed. |
| `license_module` | Whether this build can verify a license. |

The answer never carries the token, the key, the license id or the install id.

</details>

`GET /api/admin/plugins/status` takes the `admin` or `super_admin` role and
returns one row per feature in the image: whether it is `compiled`, `entitled`,
`requested` and `active`, its `phase`, and for one that is off a `reason`. A
feature off for lack of a license also carries an `upgrade_url`, a path on your
own install that opens the license page.

<details>
<summary>A status row</summary>

```json
{
  "name": "graphql",
  "compiled": true,
  "entitled": false,
  "requested": false,
  "active": false,
  "phase": "registered",
  "reason": "not granted by the capability set",
  "upgrade_url": "/admin/settings/license?plugin=graphql",
  "manifest": {
    "label": "GraphQL",
    "description": "A GraphQL API generated from your content types, with subscriptions over WebSocket and a persisted-query allowlist.",
    "category": "delivery",
    "maturity": "stable"
  }
}
```

The top level also carries `compiled` and `entitled` lists, and a `requested`
list when `LYEVE_PLUGINS` is set. `manifest.category` is one of `content`,
`delivery`, `access`, `automation`, `insight`, `operations`, `compliance`, `ai`
or `platform`, and `maturity` is `stable` or `beta`. A running feature that
serves routes lists them under `routes`, each with `method`, `pattern` and the
`group` that may call it (`public`, `auth`, `admin` or `super_admin`).

</details>

`GET /api/admin/plugins/running` answers any signed-in role with names only:
`plugins` lists what runs for the caller's tenant and `withheld` what runs on the
install but is withheld from that tenant.

**In the log**, the license decision at start is on lines that begin with
`license:`:

| Line | Meaning |
|---|---|
| `license: no key set` | `LYEVE_LICENSE_KEY` is empty. The free tier runs. |
| `license: active` | The token verified. The line carries the plan and the feature count. |
| `license: key exchanged for a signed token` | A license key was exchanged for a token. |
| `license: verification failed, checking grace cache` | The token did not verify, so the engine looks for a cached one in grace. |
| `license: running on offline grace` | A cached token is in its grace period. |
| `license: no valid token` | Nothing verified. The free tier runs. |

## Settings

| Variable | What it does | Default |
|---|---|---|
| `LYEVE_LICENSE_KEY` | The signed token or the license key. Empty runs the free tier. | empty |
| `LYEVE_LICENSE_SERVER_URL` | Where a license key is exchanged. Never contacted for a token. | `https://api.lyeve.com` |
| `LYEVE_LICENSE_CACHE_DIR` | Where the verified token and the install id are cached. | `/var/lib/lyeve` |
| `LYEVE_PLUGINS` | The features to start. Empty starts every licensed feature. | empty |

The [configuration reference](../reference/configuration.md) lists the rest.

## Related

- [Features](../features/index.md): every feature and what it does.
- [Verify offline](../trust/verify-offline.md): check that a token never calls
  home.
- [Company guide](../guides/company/index.md): licensing for evaluators.
- [Secure your instance](../how-to/secure-your-instance.md): narrowing what runs
  as a hardening step.
