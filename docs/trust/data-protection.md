---
title: Data protection
description: "Check what LyEve does for data subject requests, legal holds, retention, residency, masked personal data and tenant deletion, through routes you can call."
sidebar:
  order: 3
---

This page answers the data-protection section of a vendor questionnaire. Every
claim on it is a route or a setting, so a reviewer with a `super_admin` token
can check each one against a running install. Where a control does not exist,
the page says so. Where the admin console does the same as a route, the page
gives both paths, and the console steps assume you are signed in as a super
admin. Each feature page carries the configuration and the full request and
response shapes.

| Control | Requires |
|---|---|
| Subject access and erasure | Included free |
| Tenant deletion | Included free |
| Legal holds and audit retention | A license with the `audit-retention` feature (any license with `audit` includes it) |
| Data residency | A license with the `data-residency` feature |
| Masking personal data | A license with the `pii-mask` feature |

See [pricing](https://lyeve.com/pricing/).

## Where personal data lives

LyEve is self-hosted. Content, users, sessions, audit entries and every
feature's data live in the database you run, and none of it is sent to LyEve.
Given a signed license token, verification is an offline signature check and
the server makes no call to LyEve at all. [Verify Offline](verify-offline.md)
shows how to prove that on your own machine. Given an opaque license key
instead, the server sends that key and a random install id to the LyEve license
server to receive a token, and nothing else.

The only other traffic the server originates is traffic a feature is configured
to send (mail, webhooks, flow outbound nodes, identity providers, object
storage, the AI feature's model provider), and each is off until an
administrator turns it on. [Security Controls](security-controls.md) lists what
each one sends.

On a multi-tenant install, every query is limited to the caller's tenant. The
routes that read across tenants require `super_admin` and are documented as
such. [Tenants](../features/tenants.md) explains how a request picks its tenant.

## Subject access: export

`POST /api/admin/gdpr/export`, `super_admin`, rate limited per client address.

<div class="paths">
<div data-path="console">

Open **Settings > Privacy requests**. Under **Export subject data**, enter the address or user
id in **Subject (email or ID)** and choose **Export**. The page counts what it found, such as
**1 record from 1 of 4 plugins**, shows the bundle and offers **Download JSON**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/gdpr/export \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"identifier": "person@example.com"}'
```

</div>
</div>

`identifier` is an email address or a user id, 1 to 320 characters, with no
control characters. Every feature that stores personal data contributes its
section, and the response merges them:

| Field | Meaning |
|---|---|
| `identifier` | The identifier queried. |
| `plugins` | One section per feature that answered, keyed by section name. |
| `incomplete` | `true` when at least one section failed, so the bundle is missing data the subject is entitled to. |
| `scope` | `mode` (`tenant` or `all_tenants`), `tenants` reached, `tenants_requested`, `tenants_covered`. Always present, so a bundle says what it looked at. |
| `summary` | `total_records`, `plugins_queried`, `plugins_with_data`. |
| `errors` | Non-fatal section errors, always an array. |

On a multi-tenant install, `{"identifier": "...", "all_tenants": true}` sweeps
every tenant. The result comes back keyed per tenant under `tenants` and lists
`matched_tenants`. It is never merged flat, because the same address can name two
different people in two tenants. On a single-tenant install the sweep reaches the
`default` tenant alone, and `scope` says `tenants_covered: 1`. The sweep is API
only on v0.21.0: the console asks of your own tenant.

The export is never masked (see [Masking](#masking)), so the bundle carries the
subject's data as a subject access request requires.

**Check it:** export an address you know, then read `scope.tenants_covered` and
`incomplete` before you trust the bundle.

## Right to erasure

`POST /api/admin/gdpr/erase`, `super_admin`, rate limited, same body as export.

<div class="paths">
<div data-path="console">

On **Settings > Privacy requests**, under **Erase subject data**, enter the subject in
**Subject** and again in **Type the subject again to confirm**. **Erase permanently** stays
unavailable until the two match. The result reads **Erasure complete** with the number of rows
erased, and clears itself after a few seconds.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/gdpr/erase \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"identifier": "person@example.com"}'
```

```json
{ "identifier": "person@example.com", "total_rows": 4, "errors": null }
```

</div>
</div>

Every feature that contributes to an export also erases. Erasure removes or
anonymizes the subject's rows in each of them and answers:

| Field | Meaning |
|---|---|
| `identifier` | The identifier erased. |
| `total_rows` | Rows removed or anonymized across every feature. |
| `errors` | `["one or more erasers failed"]` when a feature failed. |
| `holds` | The active legal holds that stopped the erasure, each with `id` and `reason`. Non-empty means nothing was erased. |

An erasure by address reaches every account that address names, in every
tenant. That includes accounts a tenant's identity provider, SCIM directory or
magic link sign-up created, and what sign-in keeps about them: the SCIM
directory record, with its external id and stored attributes, and the OAuth
provider links. The account row itself stays, with its address replaced by
`erased-<user id>@localhost` and its roles cleared.

Erasure checks legal holds before it touches a row. A hold that cannot be
checked is not treated as no hold: the server answers `503` with
`legal hold status is unavailable; erasure was not attempted`, and the request
can be retried. `LYEVE_GDPR_HOLD_CHECK=proceed` changes that to erase anyway.

In the audit log, erasure redacts the subject's entries in place rather than
deleting them: the user id is cleared, the address and user agent become
`[redacted]`, and the before and after state is emptied. It finds entries by
user id or IP address, so erase by the user id to reach a person's audit
entries. An email address matches none.

An export the [data export](../features/data-export.md) feature delivered to
an S3 bucket you named is a file the instance cannot delete. For each finished
export of that kind the erasure matches, it writes the audit entry
`data_export.job.file_not_erased` with the bucket, region and object key,
filed under the export's tenant, so you can remove the file yourself. Without
an audit log, it logs a warning with the same location.

With the `audit` feature licensed, both routes write an audit entry
(`gdpr.export`, `gdpr.export.all_tenants`, `gdpr.erase`) recording the actor,
address and user agent. Each entry is filed under no tenant, so only a super
admin reading every tenant's log sees it, never a tenant's admins. The entry names the subject by a digest rather than by
the identifier, so the log proves the request happened without keeping who it
was about: `sha256:` followed by the hex SHA-256 of the identifier exactly as
the request sent it. To find the entry for a subject later, compute the digest
and filter the audit log by it. This lookup is API only on v0.21.0, because the
console's audit log search reads a resource id only when it is a UUID:

```bash
printf '%s' 'ada@example.com' | sha256sum
curl "http://localhost:3001/api/admin/audit-log?resource_type=gdpr_dsar&resource_id=sha256:<digest>" \
  -H "Authorization: Bearer $TOKEN"
```

Entries an earlier version wrote name the identifier itself. Without the
`audit` feature the server logs a warning that the operation completed with no
audit trail, so an unaudited export still leaves a trace.

## Legal holds

> **Requires** a license with the `audit-retention` feature. See [pricing](https://lyeve.com/pricing/).

Holds are part of the [Audit log](../features/audit-log.md).

| Method | Path | Role |
|---|---|---|
| `GET` | `/api/admin/legal-holds` | admin |
| `POST` | `/api/admin/legal-holds` | super_admin |
| `DELETE` | `/api/admin/legal-holds/{id}` | super_admin |

A hold has a required `name`, a `description`, and filters: `filter_action`,
`filter_resource_type`, `filter_resource_id`, `filter_actor` (a user id, which
must be a UUID), `filter_from` and `filter_to`. A hold has no expiry. It stays
in force until it is released. A released hold is kept for 30 days, so the
record that it existed outlives it, and retention enforcement keeps honoring it
for those 30 days.

Two things check holds:

- Retention enforcement skips any audit entry a hold matches.
- Subject erasure looks for active holds in the caller's tenant whose
  `filter_resource_id` or `filter_actor` equals the subject identifier, and
  refuses to erase while one exists.

**Place a hold and release it.**

<div class="paths">
<div data-path="console">

1. Open **Insight > Audit log** and choose **Retention and holds**.
2. Choose **New hold**. Enter a **Name** and **Why it was raised**, and narrow it with
   **Action**, **Resource type**, **Resource id** or **Tenant**. Leaving every filter empty
   holds the entire audit log. The form has no actor or date filters, which the API takes.
3. Choose **Create**. The hold is listed under **Legal holds**.
4. To release it, choose **Release** on its row, then **Release** in the dialog. The page
   confirms **Released** with the hold's name.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/legal-holds \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Supplier dispute", "description": "Keep what touches this customer", "filter_resource_id": "person@example.com"}'
```

The answer is `201` with the hold. With [masking](#masking) on, it shows the address as
`[redacted-email]`, and so does the list. Copy its `id` into `HOLD_ID`. Release it with
`DELETE /api/admin/legal-holds/$HOLD_ID`, which answers `204`.

</div>
</div>

While the hold stands, an erasure of `person@example.com` erases nothing. The API answers
`200` with `"total_rows": 0` and the hold under `holds`. The console reads **Erasure held**,
names the hold and says to release it first. A user's own erasure on the Content API,
under a hold whose `filter_actor` is their user id, answers `409`.

A hold does not stop `POST /api/admin/audit-log/prune`, which a super admin can
run by date.

`POST /api/admin/retention/compliance-export` (super_admin) with
`{"legal_hold_id": "...", "format": "json"}` (or `csv`) returns the entries a
hold preserves, for disclosure to a court or a regulator. It is API only on
v0.21.0.

## Retention

> **Requires** a license with the `audit-retention` feature. See [pricing](https://lyeve.com/pricing/).

Audit-log retention is a policy per `event_type` and `resource_type`:

| Method | Path | Role |
|---|---|---|
| `GET` | `/api/admin/retention-policies` | admin |
| `POST` | `/api/admin/retention-policies` | super_admin |
| `PUT` | `/api/admin/retention-policies/{id}` | super_admin |
| `DELETE` | `/api/admin/retention-policies/{id}` | super_admin |
| `POST` | `/api/admin/retention/enforce` | super_admin |

`retention_days` is the window, 365 when omitted. With `archival_enabled` the
expired entries are written to the configured storage before they are deleted,
and an archive failure keeps the rows.

**Set a policy and enforce it now.**

<div class="paths">
<div data-path="console">

1. Open **Insight > Audit log** and choose **Retention and holds**.
2. Choose **New policy**. Enter the **Event type** and **Resource type** it covers, or leave
   either empty to cover every one, and the window in **Keep for, in days**.
3. **Archive before deleting** starts on. Name the storage backend in **Archive to**, or turn
   the switch off to delete outright. The form refuses to save with archival on and no
   backend named.
4. Choose **Create**. The page confirms **Policy created**.
5. Choose **Enforce now**, then **Delete them** in the dialog. The page reports how many
   policies ran and how many entries were deleted.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/retention-policies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"event_type": "content.update", "resource_type": "content", "retention_days": 30}'
```

The answer is `201` with the policy, `"archival_enabled": false` unless you send it. Then
enforce the policies at once:

```bash
curl -X POST http://localhost:3001/api/admin/retention/enforce \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{}'
```

</div>
</div>

The `enforce` route applies the policies of the caller's tenant and reports per
policy `entries_archived`, `entries_deleted` and any `archive_error`. The body
`{"cleanup_only": true}` deletes without archiving.

A scheduled pass enforces the policies of every tenant that holds one, once at
start and then every 24 hours, or every `RETENTION_ENFORCE_INTERVAL` when set.
A failure in one tenant is logged and the pass moves on to the next. The
`enforce` route runs the caller's tenant at once.

Two other windows are fixed by configuration rather than policy: the masking
access log is kept for `PII_MASK_RETENTION_DAYS` (default 90), and a released
legal hold for 30 days.

## Residency

> **Requires** a license with the `data-residency` feature. See [pricing](https://lyeve.com/pricing/).

[Data residency](../features/data-residency.md) assigns each tenant one primary
region and records replicas and planned moves between regions. Regions and the
report need a super admin. An admin can read and set their own tenant's region.

Enforcement is on only when the engine starts with `INSTANCE_REGION` set. It
applies to writes on the Content API, and reads pass. A write for a tenant
assigned to another region is refused with `421 Misdirected Request`, the body
`{"error":"write routed to wrong region","target_region":"..."}` and the headers
`X-CMS-Region-Target` and `X-CMS-Region-Reason: data-residency-enforced`. A
tenant with no assignment passes. When the tenant's region cannot be read, the
write is refused with `503` rather than allowed. Residency decides where a
tenant's writes may land. It does not look at where a client connects from.

**Define a region and read the report.**

<div class="paths">
<div data-path="console">

Open **Settings > Data residency** and choose **New region**. Fill in **Slug**, such as
`eu-central`, **Name** and **Runs on**, such as `aws`, then choose **Create**. The report at
the top counts **Regions**, **Tenants**, **Replicated** and **Not placed**, and lists each
tenant with its region. Placing a tenant in a region is API only on v0.21.0.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/regions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"slug": "eu-central", "display_name": "EU Central (Frankfurt)", "provider": "aws"}'
```

The answer is `201` with the region. Copy its `id` into `REGION`, and the tenant's id from
`GET /api/admin/tenants` into `TENANT`. Then place the tenant:

```bash
curl -X POST http://localhost:3001/api/admin/tenants/$TENANT/region \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"region_id\": \"$REGION\"}"
```

The answer is `200` with the tenant's `region_slug`. Read the report with
`GET /api/admin/residency/report`.

</div>
</div>

The report lists every region, every tenant's assignment and a summary with
`unassigned_count`, which the console shows as **Not placed** and which a
reviewer wants at zero.

**Check it:** on an engine started with `INSTANCE_REGION`, write as a tenant
assigned to a different region and expect the `421`. Each write's answer also
carries `X-CMS-Region` with the engine's own region. The security report's
`data-residency` row reads `PASS` and names the region once the engine checks
writes.

## Masking

> **Requires** a license with the `pii-mask` feature. See [pricing](https://lyeve.com/pricing/).

[PII masking](../features/pii-masking.md) redacts personal data in text
response bodies (JSON, XML, form data, JavaScript and any `text/*` type) up to
2 MiB, and the built-in patterns also run over every log line. Past 2 MiB the rest of the body is sent unmasked,
and binary bodies and event streams pass through.

Six patterns are built in: tokens, email addresses, phone numbers, IPv4
addresses, credit card numbers (issuer prefix and Luhn checked) and US social
security numbers. A tenant can
add up to 50 enabled rules of its own. A custom pattern is at most 512
characters, compiled when it is saved, and RE2, which runs in linear time, so no
pattern can stall a response.

On the Admin API these paths are never masked, because the personal data is the
record being administered there: `/api/admin/gdpr`, `/api/admin/auth`,
`/api/admin/users`, `/api/admin/invitations`, `/api/admin/admin-tokens` and
`/api/admin/api-keys`. Reads there are still recorded. A super admin also reads
the audit log, the compliance export of a legal hold and a tenant's member list
in full, and each of those reads is recorded. Everyone else gets them masked.
Answers that hold only configuration or text the product ships are sent as
written and not recorded, as [PII masking](../features/pii-masking.md#how-it-works)
lists. The access log records `viewer_user_id`,
`viewer_role`, `pii_type` and `accessed_at` for every response that carried
personal data to a signed-in caller, kept for `PII_MASK_RETENTION_DAYS`.

Masking is a safety net. Roles, access rules and tenant scoping decide who may
read a record, and masking catches what those let through into a log or a
response. In the console, **Settings > PII masking** lists the built-in rules
in the order they run, the tenant's own rules and the count of unmasked reads.
[PII masking](../features/pii-masking.md) adds a rule both ways.

**Check it:**

<div class="paths">
<div data-path="console">

Open **Settings > Security**. The **PII masking** card reads **Enforcing** when masking is
active.

</div>
<div data-path="api">

`GET /api/admin/security/controls` shows the `pii-mask` row as `PASS` when masking is
active.

</div>
</div>

Then read any entry whose text holds an email address through the Content API
and see `[redacted-email]` in its place.

## Tenant deletion

> **Included free** on every install.

`DELETE /api/admin/tenants/{id}`, `super_admin`, signed-in session only. See
[Tenants](../features/tenants.md).

<div class="paths">
<div data-path="console">

Open **Operations > Tenants**, choose **Delete** on the tenant's row and confirm in the
dialog. The page confirms **Deleted** with the tenant's name.

</div>
<div data-path="api">

Copy the tenant's `id` from `GET /api/admin/tenants` into `TENANT_ID`, then:

```bash
curl -X DELETE http://localhost:3001/api/admin/tenants/$TENANT_ID \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

It is a hard delete, not a flag. Every row carrying the tenant is deleted in one
transaction, and the request either removes all of it or none of it. The answer
is `204 No Content`. The server refuses to start while any table that holds
tenant data has no deletion covering it, so a tenant delete never leaves rows
behind. Stored media files are left to your storage backend's own lifecycle
rules: the delete removes the tenant's media records, not the objects.

`POST /api/admin/tenants/{id}/archive` is the reversible alternative and deletes
nothing. `POST /api/admin/tenants/{id}/restore` brings an archived tenant back.
In the console, the tenant's own page has **Archive** and then **Restore**.

## Is it enforcing?

`GET /api/admin/security/controls`, `super_admin`, reports each control's status
live: `PASS` (enforcing), `WARN` (included but not configured), `FAIL`
(configured but not enforcing) or `SKIP`, with a sentence on what turns it on.
Its controls include `pii-mask` and `data-residency`. In the console,
**Settings > Security** shows the same report as cards. See
[Security Controls](security-controls.md#is-it-enforcing).

## Not yet

- **SOC 2.** LyEve has no SOC 2 report.
- **Unmasked access by role.** No setting lets a role read past the masking.
  Only a super admin does, on the audit log, a legal hold's compliance export
  and a tenant's member list.
- **Client geolocation.** Residency governs where a tenant's writes land, not
  where a request comes from.
