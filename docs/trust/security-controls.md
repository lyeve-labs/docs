---
title: Security controls
description: "Check the audit chain, API keys, rate limits, the web application firewall, sign-in controls and what the server never sends anywhere, on a running install."
sidebar:
  order: 4
---

This page answers the security section of a vendor questionnaire. Each control
is stated as the route, header or setting a reviewer can check against a running
install with a `super_admin` token, and each section ends with how to check it.
Where the admin console shows the same, the page gives both paths, and the
console steps assume you are signed in as a super admin. Where a control does
not exist, the page says so, and the last section lists what is not built.
Configuration lives on each feature's page.

| Control | Requires |
|---|---|
| Security report, API keys, rate limit protections, sign-in, TOTP and backup codes, passkey and cross-device sign-in, the super admin MFA reset, one OpenID Connect provider, setup token, password recovery | Included free |
| [Monthly request limits per API key](../features/api-key-request-limits.md) | A license with `apikey-pro` |
| Custom rate limit rules, and changes to a sign-in protection | A license with `rate-limit-pro` |
| Passkey enrollment, and marking a passkey as the backup key | A license with `mfa-pro` |
| More OpenID Connect providers, the Okta and Azure AD templates, role mapping and a backup provider | A license with `oauth-pro` |
| More than three admin seats | Any license that is active or in grace |
| Audit trail | A license with `audit` |
| [Streaming the audit trail](../features/audit-log-streaming.md) to a SIEM | A license with `audit` and `audit-pro` |
| Web application firewall | A license with `waf` |
| SAML, SCIM | A license with `saml` or `scim` |

See [pricing](https://lyeve.com/pricing/). Data subject requests, holds,
retention, residency and masking are on [Data Protection](data-protection.md).

## Is it enforcing?

`GET /api/admin/security/controls`, `super_admin`. The server evaluates each
security control at startup and logs the result. This route returns the same
report evaluated live, so a control turned on by a license change after startup
shows without a restart.

<div class="paths">
<div data-path="console">

Open **Settings > Security**. The top row counts the controls that are **Enforcing**,
**Not enforcing**, **Not configured** and **Not active**. Below it, each control has a card
with its state, what it protects against and the engine's own detail line. A card that is not
enforcing says **To turn it on** and what to do.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/security/controls \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

The answer, which the console reads:

```json
{
  "checked_at": "2026-10-02T09:10:06Z",
  "controls": [
    {"control": "pii-mask", "status": "SKIP", "detail": "pii-mask plugin not active", "remedy": "Enable a plugin that masks personal data in responses and logs."},
    {"control": "waf", "status": "PASS", "detail": "..."}
  ],
  "failures": 0
}
```

| Status | Console | Meaning |
|---|---|---|
| `PASS` | **Enforcing** | Included, configured and enforcing |
| `WARN` | **Not configured** | Included but not configured |
| `FAIL` | **Not enforcing** | Configured but not enforcing |
| `SKIP` | **Not active** | Included but not running, for example without a license |

A row other than `PASS` carries `remedy`, the sentence that turns the control
on. A control the server does not include has no row. The controls reported are
`pii-mask`, `data-residency`, `quota/rate-limit`, `quota` (usage quota
enforcement), `waf`, `mfa` and `session-anomaly` (login history and brute-force
detection from [Trusted devices](../features/trusted-devices.md)). `FAIL` is the
row that matters: a control that is configured and not enforcing. Start the
questionnaire from this response.

The `data-residency` row follows `INSTANCE_REGION`. Without it, the row reads
`WARN` and says to set it. With it, the row reads `PASS` and names the region
once the engine checks Content API writes against it. A `WARN` with
`INSTANCE_REGION` set means the engine has not started checking writes yet,
and a restart fixes it. [Data Protection](data-protection.md#residency) shows
how to prove it with a write.

## Audit trail

> **Requires** a license with the `audit` feature. See [pricing](https://lyeve.com/pricing/).

The [Audit log](../features/audit-log.md) records each entry with `sequence`,
`tenant_id`, `user_id`, `action`, `resource_type`, `resource_id`, `ip`,
`user_agent`, the timestamp, the before and after state of the record, and
`chain_hash`.

`chain_hash` is an HMAC-SHA256 over the previous entry's hash and every field
above. Altering an entry outside the API breaks its hash and every position
after it up to the next seal.
The first entry links to a fixed starting value. The key is
`LYEVE_AUDIT_HMAC_KEY` (64 hex characters). An install without one hashes with a
zero key and logs a warning at startup, and production refuses to start without
it. The chain is one sequence for the whole install, not one per tenant.

What is recorded includes:

- content create, update, delete, rollback and schedule
- schema and schema preset changes
- API key create, revoke, delete and monthly limit changes
- admin token changes
- tenant creation, changes, cloning and deletion, and domain mapping changes
- sign-in factor changes: TOTP and passkey enrollment and removal, backup codes,
  an administrator's MFA reset (`mfa.admin.reset`)
- password reset requests and completions
- rate-limit and WAF rule changes
- data export jobs
- subject export and erasure

Not recorded on v0.56.0: creating or deleting an account, changing its roles,
a super admin setting its password, granting a tenant membership, a password or SAML sign-in,
archiving a tenant and withholding a tenant's features.

How entries change or leave, so a reviewer knows what the chain can and cannot
show:

- No route edits an entry's content.
- Subject erasure redacts the person in place: it clears `user_id`, sets `ip`
  and `user_agent` to `[redacted]` and empties the before and after state. It
  leaves a signed seal for each entry it redacts, and only for an entry that
  verified first.
- Entries are deleted by `POST /api/admin/audit-log/prune` (super admin), by a
  retention policy, or by tenant deletion. Each leaves a signed seal for every
  entry it removes, so the chain stays verifiable. A legal hold stops a prune
  and a retention policy from deleting what it covers, and a tenant with an
  active hold cannot be deleted.

<details>
<summary>Audit routes</summary>

| Method | Path | Role |
|---|---|---|
| `GET` | `/api/admin/audit-log` | admin. Anyone but a super admin sees only their tenant, with `ip` and `user_agent` masked |
| `POST` | `/api/admin/audit-log/prune` | super_admin. Body `{"older_than": "<RFC3339>"}`, answers `{"deleted": n}` |
| `GET` | `/api/admin/audit-log/verify` | super_admin. Answers `valid`, `entries`, `sealed`, `broken` and `broken_sequences` |
| `POST` | `/api/admin/audit-log/rehash` | super_admin. Optional body `{"previous_key": "<64 hex>"}` after a key change |
| `POST` | `/api/admin/audit-log/export` | admin |
| `POST` | `/api/admin/audit-log/export-all` | super_admin |

</details>

In the console, **Insight > Audit log** lists the entries, filters them and
exports them as JSON. Checking the chain is API only on v0.22.0.

**Check it:** call `GET /api/admin/audit-log/verify` as a super admin and expect
`"valid": true`, with `"broken": 0` and an empty `broken_sequences`. The list route returns `chain_hash` per entry, but not the
before and after state the hash covers, so the chain cannot be recomputed
outside the server. See [Audit log](../features/audit-log.md#verify-the-chain).

## API keys

> **Included free** on every install. Monthly request limits per key need `apikey-pro`.

A key is `ly_` followed by 64 hex characters from 32 random bytes. The plaintext
is returned once, in the `raw_key` field of the create response, and never
stored. The server keeps an HMAC-SHA256 of the key under its pepper (plain
SHA-256 when no pepper is configured), and never returns it. See
[API keys](../features/api-keys.md).

A key carries `roles`, `schemas`, `scopes` (`resource:action`), `expires_at` and
an optional `monthly_limit`. A key that holds the `admin` or `super_admin` role
must expire, at most 90 days ahead. A key's roles cannot exceed its creator's,
unless the creator is a super admin. Revoke disables the key in place. Delete
removes it with its usage history. A disabled or expired key authenticates
nothing. Clients send the key in the `X-API-Key` header, and every use is
recorded in the key's history.

A key's roles go through the same access rules as a person's, field masks
included. A key with a `schemas` list is refused every other content type with
`403`, whatever its roles allow.

There is no rotate route. Rotation is: create the replacement, deploy it, revoke
the old key. SCIM provisioning tokens do rotate in place
(`POST /api/admin/scim/providers/{id}/rotate`).

<details>
<summary>API key routes</summary>

| Method | Path | Role |
|---|---|---|
| `GET`, `POST` | `/api/admin/api-keys` | admin |
| `POST` | `/api/admin/api-keys/{id}/revoke` | admin |
| `DELETE` | `/api/admin/api-keys/{id}` | admin |
| `GET` | `/api/admin/api-keys/{id}/audit` | admin |
| `PATCH` | `/api/admin/api-keys/{id}/monthly-limit` | admin |

</details>

**Check it:** create a key on **Access > API keys** or with `POST /api/admin/api-keys`, use
it once, and read `GET /api/admin/api-keys/{id}/audit`. Each use is a row with its method,
path, status and address. A key's history is API only on v0.22.0.

## Rate limits

> **Included free** on every install. Custom rules and changes to a sign-in protection need the `rate-limit-pro` feature.

Every limit answers `429` with `Retry-After`, `RateLimit-Limit`,
`RateLimit-Remaining`, `RateLimit-Reset`, `RateLimit-Policy` and the
`X-RateLimit-*` equivalents.

**The built-in public limiter** is always on. It covers the public routes of
features, `POST /api/v1/auth/token`, `POST /api/admin/auth/login`, `/refresh`
and `/mfa-verify`, and the two subject request routes. First-run setup
(`/api/admin/setup`) is outside it. Every client address shares one cap of 50
requests per second with a burst of 100 across all of these routes, and these
routes also have their own budget per address:

| Route | Rate per second | Burst |
|---|---|---|
| Login, token refresh, MFA step and Content API token | 5 | 10 |
| Password reset request | 3 | 5 |
| Password reset confirm | 5 | 10 |
| Magic link request | 5 | 10 |
| Magic link verify | 10 | 20 |
| Subject export | 10 | 20 |
| Subject erasure | 5 | 10 |

`PUBLIC_RATE_LIMITS` (`METHOD:/pattern=rate:burst,...`) and
`PUBLIC_RATE_LIMIT_GLOBAL` (`rate:burst`) override them, and a malformed entry
stops the server at startup. The Admin API and the Content API each keep their
own limiter, each tracking at most 10,000 address and route pairs and dropping
the least recently used.

**The global limiter** is `RATE_LIMIT_RPS`, a per-address limit on every
request. Production refuses to start without it. Counts are in memory per
instance, or shared across replicas with `RATE_LIMIT_BACKEND=redis`, which is
free.

**The sign-in protections**, managed under
[Rate limiting](../features/rate-limiting.md): `POST /api/v1/auth/token` and
`POST /api/admin/auth/login` each allow 5 requests per 15 minutes with a burst
of 5, plus the password reset, magic link and MFA limits. A protection can never
be set to zero requests. Custom rules per endpoint pattern, tenant or role, and
changes to a protection, need `rate-limit-pro`. A rule keeps enforcing after a
license lapses. Only editing it is refused.

The client address comes from the connection unless `TRUSTED_PROXIES` names the
proxy in front.

**Check it:** send six logins in a row for a test account and read the `429`
and its `Retry-After`. The protection counts per address, so use an address and
an account you can afford to lock for 15 minutes.

## Web application firewall

> **Requires** a license with the `waf` feature. See [pricing](https://lyeve.com/pricing/).

The [web application firewall](../features/web-application-firewall.md)
inspects the path and query (URL-decoded repeatedly), the body, the headers and
the cookies of each request against rules in the categories `sqli`, `xss`,
`path_traversal`, `tenant_injection`, `command_injection`, `ssrf` and `custom`.
A rule's action is `block` or `log`, with a severity. A request is refused when a
block rule matches or the summed anomaly score exceeds `anomaly_threshold`
(default 5). The answer is `403` with
`{"error":"request blocked by WAF","code":"waf_blocked", ...}` and the header
`X-WAF-Block: true`, and the violation is stored.

Deliberate exemptions a reviewer should know:

- Body inspection stops at `max_inspect_body_bytes` (default 128 KiB), so a
  large upload cannot stall the server.
- The bodies of content writes (`/api/admin/content`, `/api/v1/content/*` on
  POST, PUT and PATCH) are not inspected, because a CMS has to store an article
  about SQL injection. Their path, query and headers still are.
- Everything under `/api/admin/auth/`, the setup routes and the firewall's own
  `/api/admin/waf/` routes is skipped, so a false positive cannot lock every
  administrator out.

False positives are recorded per rule, path pattern and parameter, for one
tenant or the whole install.

<details>
<summary>Firewall routes</summary>

| Method | Path | Role |
|---|---|---|
| `GET`, `POST` | `/api/admin/waf/rules` | admin |
| `GET`, `PUT` | `/api/admin/waf/rules/{id}` | admin |
| `GET`, `DELETE` | `/api/admin/waf/violations` | admin |
| `GET`, `POST` | `/api/admin/waf/false-positives` | admin |
| `DELETE` | `/api/admin/waf/false-positives/{id}` | admin |
| `GET` | `/api/admin/waf/config` | admin |
| `PUT` | `/api/admin/waf/config` | super_admin |
| `GET` | `/api/admin/waf/stats` | admin |

</details>

**Check it:** request `/api/v1/content/post?q=1%27%20OR%201=1--` and read the
`403` and its `X-WAF-Block` header.

## Sign-in

> **Included free** on every install. Enrolling a passkey needs `mfa-pro`, more than one OpenID Connect provider needs `oauth-pro`, and SAML and SCIM need the `saml` and `scim` features.

Passwords are hashed with bcrypt by default, or argon2id with
`PASSWORD_HASH_ALGO=argon2id`. Sessions are tokens signed with Ed25519, key at
`JWT_KEY_PATH`, lifetime `JWT_EXPIRY_SECS` (default 900, at most 3600 in
production), with refresh tokens lasting `REFRESH_TOKEN_TTL_SECS` (default 30
days). When the key file cannot be written, production refuses to start and
development signs with HMAC-SHA256 over `JWT_SECRET` instead.

After five failed logins in 15 minutes the account is locked for the rest of the
window. MFA codes have their own limits: five failed codes per user in 15
minutes, five attempts per challenge, and a challenge that verifies once cannot
be used again. With [Caching](../features/caching.md) on `CACHE_DRIVER=redis`
and `REDIS_URL` set, this state is shared: replicas enforce one limit between
them, a restart does not lift a lock, and a used challenge is refused on every
replica. Without it the state is per instance, and the server says which at
startup. If Redis cannot be read, a login is refused and an MFA challenge is not
accepted.

**First-run setup.** `POST /api/admin/setup` is public because no account exists
yet, so creating the first `super_admin` demands a setup token. It is
`LYEVE_SETUP_TOKEN` when set (16 characters or more, or the server does not
start), otherwise a random token the server prints to its log once at startup
while no account exists, so reading it proves access to the host. Only a digest
is kept, and a presented token is compared in constant time. A missing or wrong
token answers `401`. When two claims arrive together exactly one wins, and
creating the account retires the token. Once an account exists, setup answers
`409`.

**MFA.** [Multi-factor authentication](../features/multi-factor-auth.md) offers
TOTP and single-use backup codes on every install, and WebAuthn passkeys
(resident keys and user verification preferred) with passwordless and
cross-device login. Enrolling a passkey needs `mfa-pro`. A passkey enrolled
under it keeps signing in after a license lapses. Routes are under
`/api/admin/mfa/` for enrolled users and `/api/admin/auth/webauthn/` for login.
TOTP secrets are encrypted with a per-tenant key derived from `ENCRYPTION_KEY`
(HKDF-SHA256, AES-256-GCM). The server does not start without that key, and it
must differ from `JWT_SECRET`. Backup codes are stored as SHA-256 hashes. `POST /api/admin/mfa/admin-reset` disables a
user's MFA, requires a reason, needs a super admin not scoped to a
tenant, and is audited as `mfa.admin.reset` under the tenant the reset account
belongs to. A login answers
`mfa_required: true` when the user has a factor enrolled. There is no setting
that makes MFA mandatory for a role or a tenant. Enrollment is per user.

**Single sign-on.** [OAuth sign-in](../features/oauth-sign-in.md) does OpenID
Connect by discovery with nonce and JWKS validation of the id token. One
provider is free, from the Google, GitHub and Apple templates or by hand. More
providers, the Okta and Azure AD templates, a custom roles claim or default
roles, and a backup provider need `oauth-pro`. [SAML SSO](../features/saml-sso.md) is an SP-initiated SAML 2.0
service provider with templates for Okta, Azure AD and OneLogin, just-in-time
user creation, a role mapping from the assertion (default `editor`) and
`POST /api/admin/saml-providers/{id}/rollover-cert` for certificate rotation.
[SCIM provisioning](../features/scim-provisioning.md) serves SCIM 2.0 Users and
Groups at `/api/v1/scim/v2/` behind a bearer token per provider, with PATCH and
filtering.

:::caution[Known defects]
Today the Admin API refuses the form post an identity provider sends to the
SAML assertion consumer route, and the Content API refuses SCIM requests sent as
`application/scim+json`. Both answer `415`, so an identity provider that uses
either cannot complete SAML sign-in or SCIM provisioning yet.
:::

**Recovering an account.** Two paths, both free.
[Password reset](../features/password-reset.md) answers `200` whether or not the
address exists, sets the new password from a single-use token that lasts one
hour, requires twelve characters or more, and ends the user's refresh tokens. Or
a super admin sets the password, with the **Set password** action on the Users
page or `PUT /api/admin/users/{id}/password` and `{"password": "..."}`. The new
password is checked against the install's password policy, and on success every
session the account held ends. [Magic link sign-in](../features/magic-link-sign-in.md)
is the passwordless email login beside these.

## What the server never sends anywhere

There is no telemetry channel to LyEve. Given a signed license token,
verification is an offline signature check and the server makes no call to
LyEve. [Verify Offline](verify-offline.md) shows how to confirm that with a
network namespace, strace and a firewall.

The one call to LyEve happens only for an opaque license key
(`lyeve_live_...`), set in `LYEVE_LICENSE_KEY` or posted to
`POST /api/admin/license/renew`. The server posts the key and its random install
id over https to `LYEVE_LICENSE_SERVER_URL`, verifies the token it gets back
against the public key built into the server, caches it, and exchanges again
only when the cached token is within seven days of expiry or no token is
active. A key the server
refuses grants nothing. When the server cannot be reached, a cached token keeps
working. See [Licensing and tiers](../concepts/licensing-and-tiers.md).

Content leaves the install only through features an administrator configures,
each to a destination that administrator chose:

- [Webhooks](../features/webhooks.md) send the record's data, and its previous
  data on an update, to the URLs you register.
- [Email](../features/email.md) and the outbound nodes of
  [flows](../features/flows.md) send what you put in them.
- The [AI](../features/ai.md) feature sends the request to the model provider
  you configured: the system prompt, the conversation (which includes the record
  being edited and its title), and for an image operation the picture. The
  request carries no tenant identifier, and nothing is redacted before it is
  sent. A local endpoint on your own network sends nothing to a third party.
  Transcripts are stored in your database under the retention window you set,
  and are covered by tenant deletion and by subject export and erasure.

## Where outbound requests may go

Webhooks, flows, identity providers, JWKS fetches and alert channels resolve
the host they call and refuse an address in any of these ranges:

| Range | What it is |
|---|---|
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Private networks |
| `100.64.0.0/10` | Shared address space, used by carrier NAT, Tailscale and some cloud metadata services |
| `127.0.0.0/8`, `::1/128` | Loopback |
| `169.254.0.0/16`, `fe80::/10` | Link-local, where cloud metadata services answer |
| `0.0.0.0/8`, `::/128` | This network, and the unspecified address |
| `224.0.0.0/4`, `ff00::/8` | Multicast |
| `240.0.0.0/4` | Reserved |
| `fc00::/7` | IPv6 unique local |
| `::/96`, `::ffff:0:0:0/96` | IPv6 forms that carry an IPv4 address |
| `64:ff9b:1::/48` | Local-use NAT64 |

A NAT64 address under `64:ff9b::/96`, or a 6to4 address under `2002::/16`, is
judged by the IPv4 address it carries, so `64:ff9b::a9fe:a9fe` is refused as
`169.254.169.254` while the NAT64 form of a public address passes. A
network-specific NAT64 prefix is not recognized, so its addresses are judged as
plain IPv6.

An install that reaches its own services over Tailscale sets
`LYEVE_OUTBOUND_ALLOW_CGNAT=true` to allow `100.64.0.0/10` for every tenant.
[Email](../features/email.md) runs its own check and reads
`SMTP_ALLOWED_PRIVATE_NETWORKS` instead, so a relay in that range needs
`100.64.0.0/10` listed there.

## Not yet

- **SOC 2.** LyEve has no SOC 2 report.
- **Recomputing the audit chain yourself.** The list route does not return the
  before and after state the hash covers, so only the verify route can check it.
- **API key rotation in place.** Create, deploy, revoke.
- **Account changes in the audit trail.** Creating or deleting an account,
  changing its roles or memberships, a super admin setting its password, and a
  password or SAML sign-in leave no audit entry.
- **Mandatory MFA per role or tenant.** Enrollment is per user.
