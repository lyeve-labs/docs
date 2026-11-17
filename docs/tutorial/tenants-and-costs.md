---
title: "22. Tenants and costs"
description: "Give a partner agency a tenant of its own, choose what it may use and who works in it, read what each tenant used, cap it with a quota, and track what the usage costs."
sidebar:
  order: 22
covers:
  console:
    - "/admin/tenants"
    - "/admin/tenants/[id]"
    - "/admin/tenants/usage"
    - "/admin/tenants/costs"
    - "/admin/tenants/costs/budgets"
    - "/admin/tenants/costs/prices"
  api:
    - "GET /api/admin/tenants/provisioning"
    - "POST /api/admin/tenants"
    - "PUT /api/admin/tenant-features/{tenant}"
    - "PUT /api/admin/users/{id}/memberships"
    - "POST /api/admin/tenants/{id}/archive"
    - "POST /api/admin/tenants/{id}/restore"
    - "GET /api/admin/usage/tenant/{id}"
    - "PUT /api/admin/quotas/{tenantID}"
    - "POST /api/admin/cost-monitor/aggregate"
    - "GET /api/admin/cost-monitor/dashboard"
    - "POST /api/admin/cost-monitor/budgets"
    - "PUT /api/admin/cost-monitor/prices/{resource_type}"
verified:
  engine: v0.62.0
  console: v0.28.0
---

A tenant is a separate set of content, users and settings on one install. In this lesson a
partner agency gets a tenant of its own for its trips, and you see what every tenant uses and
what that costs. You need a super admin account, and a token from signing in as that account
in `TOKEN` for the API steps, because several tenant routes take a signed-in session and refuse
an admin token. [Lesson 21](ai.md) is the one before this.

Each section starts with what it needs. A free install runs one tenant, `default`, and every
screen in this lesson except creating a tenant and setting a quota.

## See the tenants on this install

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Open **Operations > Tenants**. Only a super admin sees this entry. Each row shows a tenant's
**Slug**, **Name**, **Plan**, **Status** and **Created** date, with buttons to edit, disable or
delete it. The search box and the status filter narrow the current page.

</div>
<div data-path="api">

Ask whether this install may create tenants:

```bash
curl http://localhost:3001/api/admin/tenants/provisioning \
  -H "Authorization: Bearer $TOKEN"
```

A licensed install answers `{"enabled": true}`, and a free one `{"enabled": false}`.

</div>
</div>

On a free install the page lists `default`, has no **New tenant** button, and shows
**Tenant provisioning** with **Not enabled on this engine** above the list.

![The Tenants page on a free install, with the default tenant and the tenant provisioning card that says it is not enabled.](./media/tenants-and-costs-tenants-free.png)

![The Tenants page on a licensed install, with the New tenant button and two tenants.](./media/tenants-and-costs-tenants-licensed.png)

## Create a tenant for a partner agency

> **Requires** a license with the `multitenant-provisioning` feature. See [pricing](https://lyeve.com/pricing/).

<div class="paths">
<div data-path="console">

1. On **Tenants**, choose **New tenant**.
2. Enter `partner_agency` in **Slug** and `Partner agency` in **Display name**. Leave **Plan**
   empty.
3. Choose **Create**. The tenant joins the list.

A slug takes lowercase letters, digits and underscores, starts with a letter, and cannot change
later.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/tenants \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"slug": "partner_agency", "name": "Partner agency"}'
```

The answer is `201` with the tenant. Copy its `id` into `TENANT_ID`.

</div>
</div>

![Opening New tenant, entering the slug partner_agency and the display name, and creating it.](./media/tenants-and-costs-new-tenant.webm)

The new tenant starts empty and does not see the `default` tenant's `trip` entries.
[Run more than one tenant](../features/tenants.md#run-more-than-one-tenant)
says how a request picks its tenant.

## Choose what the tenant may use

> **Included free** on every install.

Withholding a feature from one tenant leaves the rest of the install as it was. Here the partner
agency gets no AI.

<div class="paths">
<div data-path="console">

1. On **Tenants**, choose the tenant's name, `Partner agency`. Its page opens.
2. Under **Features**, remove `ai` from **Available to this tenant**.
3. Choose **Save**. The page confirms that the features apply from the tenant's next request.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/tenant-features/partner_agency \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"withheld": ["ai"]}'
```

The answer lists `withheld` and `known`, every feature name the install can withhold.

</div>
</div>

A request in that tenant to a withheld feature is refused, and its screens leave that tenant's
console. Only features this install runs can be turned on.

## Add a member

> **Included free** on every install.

An account belongs to the tenant it was created in. A membership gives it roles in another one.

<div class="paths">
<div data-path="console">

1. On the tenant's page, under **Members**, choose **New member**.
2. Enter an existing account's address in **Account email**.
3. Under **Roles in this tenant**, tick **editor** and untick **viewer**.
4. Choose **Create**. The account is listed with its roles and **Membership** under **Access**.

</div>
<div data-path="api">

Copy the account's `id` from `GET /api/admin/users` into `USER_ID`, then:

```bash
curl -X PUT http://localhost:3001/api/admin/users/$USER_ID/memberships \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tenant_id": "partner_agency", "roles": ["editor"]}'
```

</div>
</div>

![Opening New member on the tenant's page, entering an account, choosing editor and creating the membership.](./media/tenants-and-costs-new-member.webm)

A free install holds up to three admin seats, so a membership with the `admin` role can be
refused there. [Give an account more than one tenant](../features/tenants.md#give-an-account-more-than-one-tenant)
says how the account then signs in to the tenant.

## Archive and restore a tenant

> **Included free** on every install.

Archiving makes a tenant read-only and keeps its data.

<div class="paths">
<div data-path="console">

On the tenant's page, choose **Archive**, then **Archive** again in the dialog that asks
**Archive this tenant?**. Reads still work, and every change is refused. Choose **Restore** to
make it writable again.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/tenants/$TENANT_ID/archive \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/tenants/$TENANT_ID/restore \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

## Read usage and set a quota

> **Included free** on every install. Setting a quota needs a license with the `usage-pro` feature. See [pricing](https://lyeve.com/pricing/).

Every tenant's API calls, storage and bandwidth are metered. A quota caps them.

<div class="paths">
<div data-path="console">

1. Open **Operations > Quotas and usage**. The counts at the top say how many quotas are set,
   how many tenants are blocked and how many requests for more are waiting.
2. Choose **New quota** and enter `partner_agency` in **Tenant**.
3. Enter `100000` in **API calls a month**. `0` means no limit.
4. Turn on **Hard limit** to enforce the limits rather than only report them, and
   **Block the tenant past a limit** to refuse its requests once it is over. Keep
   **Warn at 80%** and **Warn at 90%**.
5. Choose **Create**. The page confirms **Quota saved for partner_agency.**

**Requests for more** lists the higher limits tenants asked for, with buttons to approve or deny
each one.

</div>
<div data-path="api">

Read one tenant's usage for a month:

```bash
curl "http://localhost:3001/api/admin/usage/tenant/default?period=2026-11" \
  -H "Authorization: Bearer $TOKEN"
```

Set the quota:

```bash
curl -X PUT http://localhost:3001/api/admin/quotas/partner_agency \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"requests_limit": 100000, "is_hard_limit": true, "block_on_exceeded": true}'
```

</div>
</div>

On a free install, **Create** is refused with a note that raising a limit or turning on
enforcement needs a license, and the limit fields turn off. Lowering a limit to `0` and turning
enforcement off stay allowed.

![The New quota form on a free install after Create, with the license note and the limit fields turned off.](./media/tenants-and-costs-quota-free.png)

![The New quota form on a licensed install, with a monthly API call limit, a hard limit and blocking on.](./media/tenants-and-costs-quota-licensed.png)

## See what this month costs

> **Included free** on every install.

The cost ledger prices what the tenant you are signed in to uses: database, storage,
bandwidth, compute and AI. It reads the meters every hour.

<div class="paths">
<div data-path="console">

1. Open **Operations > Tenant costs**. **This month** shows **Spend so far**,
   **Budgets at risk**, **Open anomalies** and **Could save**, then the split **By resource**.
2. Choose **Update costs** to read the meters now instead of waiting for the hour. The page
   says **Costs updated**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/cost-monitor/aggregate \
  -H "Authorization: Bearer $TOKEN"

curl http://localhost:3001/api/admin/cost-monitor/dashboard \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![The This month tab of Tenant costs, with the spend so far and the split by resource.](./media/tenants-and-costs-this-month.png)

## Set a budget

> **Included free** on every install.

A budget publishes the `cost.budget.exceeded` event when spend crosses one of its thresholds,
so a [flow](../features/flows.md) can tell someone.

<div class="paths">
<div data-path="console">

1. On **Tenant costs**, choose the **Budgets** tab, then **New budget**.
2. Enter `Monthly cap` in **Name** and `25.00` in **Amount**. Keep **USD** in **Currency** and
   **Monthly** in **Period**.
3. Leave **Resource** at every resource, and pick the month in **First day** and **Last day**.
4. Keep the **Alert thresholds** and choose **Create**. The page confirms **Budgets saved.**

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/cost-monitor/budgets \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Monthly cap", "period": "monthly", "amount": "25.00", "currency": "USD", "alert_thresholds": [50, 80, 100], "period_start": "2026-11-01T00:00:00Z", "period_end": "2026-12-01T00:00:00Z"}'
```

The answer is `201` with the budget and its `current_spend`.

</div>
</div>

![Opening New budget, entering a name, an amount and the month's window, and creating it.](./media/tenants-and-costs-new-budget.webm)

## Set your own prices

> **Included free** on every install.

The built-in prices are placeholders, so replace them with what each resource costs you.

<div class="paths">
<div data-path="console">

1. On **Tenant costs**, choose the **Prices** tab. Each resource has a card marked **Default**
   or **Set by this tenant**.
2. On **Compute**, enter `2.00` in **Price** and `1000000` in **Per units**, which counts
   requests, and keep `USD` in **Currency**.
3. Choose **Save**. The page says **Price saved.**

**Use default** on a card removes this tenant's price.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/cost-monitor/prices/compute \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"unit_price": "2.00", "per_units": 1000000, "currency": "USD"}'
```

</div>
</div>

## Next

[23. Logs, errors and alerts](logs-and-errors.md) shows what the engine is doing and tells you
when it fails. For depth, read [Tenants](../features/tenants.md),
[Usage and quotas](../features/usage-and-quotas.md) and
[Tenant quotas](../features/tenant-quotas.md).
