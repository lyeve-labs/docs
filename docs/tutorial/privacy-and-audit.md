---
title: "18. Privacy, compliance and the audit log"
description: "Export or erase one person's data, choose what an erasure removes and what people export themselves, mask personal data, place a tenant in a region, and read, keep and stream the audit log."
sidebar:
  order: 18
covers:
  console:
    - "/admin/gdpr"
    - "/admin/settings/privacy"
    - "/admin/settings/pii"
    - "/admin/settings/pii/reads"
    - "/admin/settings/residency"
    - "/admin/audit-log"
    - "/admin/audit-log/retention"
    - "/admin/audit-log/sinks"
  api:
    - "POST /api/admin/gdpr/export"
    - "POST /api/admin/gdpr/erase"
    - "PUT /api/admin/schemas/form-erasure"
    - "PUT /api/admin/analytics/gdpr/export-sections"
    - "POST /api/admin/pii/rules/test"
    - "POST /api/admin/pii/rules"
    - "GET /api/admin/pii/access-log"
    - "POST /api/admin/regions"
    - "GET /api/admin/audit-log"
    - "POST /api/admin/audit-log/prune"
    - "POST /api/admin/retention-policies"
    - "POST /api/admin/legal-holds"
    - "POST /api/admin/audit-log/sinks"
verified:
  engine: v0.62.0
  console: v0.28.0
---

A person who wrote you about a trip can ask what you hold about them, or ask you to erase it.
In this lesson you answer both requests, decide what an erasure removes, hide personal data in
responses, and read the record of who changed what. Sign in as a super admin, and keep a
session token in `TOKEN` as in [lesson 17](sign-in-and-sso.md).

## Export one person's data

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Open **Settings > Privacy requests**.
2. Under **Export subject data**, enter the person's address or user id in
   **Subject (email or ID)** and choose **Export**.
3. The page counts what it found and offers **Download JSON**.

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

## Erase one person's data

> **Included free** on every install.

Erasure removes or anonymizes the person's data in every feature, and it cannot be undone.

<div class="paths">
<div data-path="console">

1. On **Settings > Privacy requests**, under **Erase subject data**, enter the person in
   **Subject (email or ID)**.
2. Enter it again in **Type the subject again to confirm**. **Erase permanently** stays
   unavailable until the two match.
3. Choose **Erase permanently**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/gdpr/erase \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"identifier": "person@example.com"}'
```

`200` means every part was erased. `207` means part of it did not finish, and running it
again finishes it.

</div>
</div>

![Typing the subject twice, the erase button turning available, and the result.](./media/privacy-and-audit-erase.webm)

When part of the data cannot be told apart from another account's data in this tenant, the
page says so and erases nothing there. **Before a person exports their own data**, lower on
the page, sets what every person must confirm before they download a copy of their own data.

## Choose what erasure and export cover

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Open **Settings > Privacy**.

1. Under **Form submissions on erasure**, choose how much of a form submission an erasure
   removes, because one submission can hold details of more than one person. Choose **Save**.
2. Under **Exporting their own data**, tick the sections in **What a person may include**
   and the roles in **Roles that may export**. Choose **Save** under each.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/schemas/form-erasure \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mode": "soft"}'
```

The export sections and roles are set at `/api/admin/analytics/gdpr/export-sections` and
`/api/admin/analytics/gdpr/export-roles`, as
[Analytics](../features/analytics.md#choose-what-the-tenant-offers-and-who-may-export) shows.

</div>
</div>

If no section is ticked, nobody in this tenant can export their own data. An administrator
can always export.

## Mask personal data in responses

> **Requires** a license with the `pii-mask` feature.

Built-in rules mask email addresses, phone numbers, IP addresses, card numbers and tokens in
responses and logs. You add rules for identifiers only you use.

<div class="paths">
<div data-path="console">

1. Open **Settings > PII masking** and choose **New rule**.
2. Enter `BOOKING_REF` in **Name** and `BK-[0-9]{6}` in **Pattern**, in RE2 syntax.
3. Enter `[redacted-booking]` in **Replaced with**.
4. Under **Try it**, paste a sample in **Sample** and choose **Run it**. **What a reader would
   get** shows the masked text. Nothing here is stored.
5. Turn on **Masking** and choose **Create**. A new rule starts off, and turning it on changes
   every response from the next request.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/pii/rules/test \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pattern": "BK-[0-9]{6}", "replacement": "[redacted-booking]", "sample": "Booking BK-004211 confirmed"}'

curl -X POST http://localhost:3001/api/admin/pii/rules \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "BOOKING_REF", "pattern": "BK-[0-9]{6}", "replacement": "[redacted-booking]", "enabled": true}'
```

</div>
</div>

![Writing a rule, testing it on a sample, and creating it with masking on.](./media/privacy-and-audit-pii-rule.webm)

**Priority** decides which rule wins when two patterns match the same text, and lower runs
first. A rule named like a preset replaces that preset. The **Unmasked reads** tab lists who
read data unmasked, and when.

<div class="paths">
<div data-path="console">

Open the **Unmasked reads** tab on **Settings > PII masking**.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/pii/access-log?limit=5" \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

On an install without the feature, **PII masking** says it is not included and responses are
not masked.

![The PII masking page on a free install, with the not enabled card.](./media/privacy-and-audit-pii-free.png)

![The PII masking page on a licensed install, with BOOKING_REF and the presets.](./media/privacy-and-audit-pii-licensed.png)

## Place tenants in a region

> **Requires** a license with the `data-residency` feature.

Only a super admin sees this screen, because it covers every tenant.

<div class="paths">
<div data-path="console">

1. Open **Settings > Data residency** and choose **New region**.
2. Enter `eu-west-1` in **Slug**, a **Name** for the report and the platform in **Runs on**.
   A slug cannot be changed later.
3. Keep **Accepting new tenants** on, and turn on **Default for new tenants** if new tenants
   should start here. Choose **Create**.

**Where tenants are** counts the regions, the tenants, the replicated ones and the ones not
placed, and lists each tenant with its region and replica.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/regions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"slug": "eu-west-1", "display_name": "EU West (Ireland)", "provider": "aws"}'
```

</div>
</div>

[Data residency](../features/data-residency.md) assigns a tenant to a region and moves it.

## Read the audit log

> **Requires** a license with the `audit` feature.

Every change to content, content types and settings is recorded as it happens.

<div class="paths">
<div data-path="console">

1. Open **Insight > Audit log**.
2. Search by action, resource type or user id, pick an **Action**, and set **From** and
   **To** to narrow the dates.
3. **Export JSON** downloads what is listed, and **Export all** downloads every entry.

**Prune** deletes the entries older than a date you pick. They cannot be recovered or
exported afterward.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/audit-log?resource_type=schema&limit=10" \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/audit-log/prune \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"older_than": "2025-01-01T00:00:00Z"}'
```

</div>
</div>

Search matches whole values only, so try the exact name of an action.

![The audit log on a free install, with the not enabled card.](./media/privacy-and-audit-audit-free.png)

![The audit log on a licensed install, filtered to the trip content type.](./media/privacy-and-audit-audit-licensed.png)

## Keep entries for a set time, or hold them

> **Requires** a license with the `audit` feature.

A retention policy deletes entries after a number of days. A legal hold keeps matching entries
past every policy until you release it. Only a super admin changes either.

<div class="paths">
<div data-path="console">

1. On **Insight > Audit log**, open the **Retention and holds** tab and choose **New policy**.
2. Leave **Event type** empty, enter `user` in **Resource type** and `180` in
   **Keep for, in days**.
3. **Archive before deleting** starts on. Name the storage in **Archive to**, or turn it off
   to delete outright. Choose **Create**.
4. To hold entries back, choose **New hold**, enter a **Name** and **Why it was raised**,
   narrow it with the filters, and choose **Create**.

**Enforce now** runs the policies at once, after a question.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/retention-policies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"event_type": "", "resource_type": "user", "retention_days": 180, "archival_enabled": true, "archival_path_prefix": "audit-archive/"}'

curl -X POST http://localhost:3001/api/admin/legal-holds \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Booking dispute", "filter_resource_type": "user", "filter_from": "2026-01-01T00:00:00Z"}'
```

</div>
</div>

A hold with every filter empty holds the entire audit log, and the page warns that no policy
deletes anything while it stands.

## Stream the log to a security tool

> **Requires** a license with the `audit-pro` feature.

<div class="paths">
<div data-path="console">

1. On **Insight > Audit log**, open the **Streaming** tab and choose **New destination**.
2. Enter a **Name**, pick the **Destination** type and enter its **URL**.
3. Turn on **Send the existing log too** to send past entries as well as new ones, then
   choose **Create**. An HTTPS destination shows its signing secret once.
4. Choose **Send a test** to check that the destination answers.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/audit-log/sinks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "security-siem", "kind": "https", "url": "https://siem.example.com/audit"}'
```

</div>
</div>

Every entry is delivered at least once and in order. The destination type cannot be changed
afterward, so create a new destination instead.

![Creating an HTTPS destination, copying its signing secret, and sending a test.](./media/privacy-and-audit-streaming.webm)

## Next

[19. Jobs and features](jobs-and-features.md) shows what runs on a schedule and which features
serve your tenant. For depth, read [Data protection](../trust/data-protection.md),
[PII masking](../features/pii-masking.md), [Data residency](../features/data-residency.md),
[Audit log](../features/audit-log.md) and
[Audit log streaming](../features/audit-log-streaming.md).
