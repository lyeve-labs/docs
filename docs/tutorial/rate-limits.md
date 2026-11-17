---
title: "20. Rate limits"
description: "Read the limits that protect your engine, change the global limit, write a rule for the trip routes, watch who is near a limit, and give your tenant its own limit and address lists."
sidebar:
  order: 20
covers:
  console:
    - "/admin/settings/rate-limits"
    - "/admin/settings/rate-limits/live"
    - "/admin/settings/rate-limits/tenant"
    - "/admin/settings/rate-limits/tenant/refusals"
  api:
    - "GET /api/admin/rate-limits/overview"
    - "PUT /api/admin/rate-limits/global"
    - "POST /api/admin/rate-limits"
    - "GET /api/admin/rate-limits/status"
    - "PUT /api/admin/rate-limits/tenant-global"
    - "POST /api/admin/rate-limits/ip-rules"
    - "GET /api/admin/rate-limits/history"
verified:
  engine: v0.62.0
  console: v0.28.0
---

Once your trips are public, a single client can call the API faster than you want. In this
lesson you read the limits that already apply, then tighten them for the trip routes and for
your tenant. Sign in as a super admin, and keep a token in `TOKEN` as in
[lesson 19](jobs-and-features.md). A tenant admin sees only the **Tenant controls** part, as
**Settings > Rate limits**.

## Read the limits that apply

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Open **Settings > Rate limits**. The **Limits** tab counts the custom rules, the enforced ones
and the callers at their limit, then lists every limit:

| Part | What it holds |
|---|---|
| **Global limit** | Every request, per address, across every tenant, with its **Rate**, **Burst** and **State** |
| **Sign-in and reset protections** | Built-in limits on sign-in and password reset, enforced on every engine, each with its **Shipped value** |
| **Custom rules** | Your own limits for an endpoint, a tenant or a role |
| **Engine limits** | Limits set in the engine's environment. Change them there, then restart the engine |

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/rate-limits/overview \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

## Change the global limit

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. On the **Global limit** row, choose **Edit the global limit**.
2. Enter `50` in **Requests per second** and `100` in **Burst**.
3. Turn on **Enforce the global limit** and choose **Save**.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/rate-limits/global \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"rate": 50, "burst": 100, "enabled": true}'
```

</div>
</div>

**Rate** is how fast the allowance refills, and **Burst** is how many requests may arrive at
once.

## Limit the trip routes for visitors

> **Requires** a license with the `rate-limit-pro` feature.

<div class="paths">
<div data-path="console">

1. Under **Custom rules**, choose **New rule**.
2. In **Choose a route**, pick `GET /api/v1/content/{schema}`. It fills in **Endpoint**.
   Change `{schema}` to `trip`, so the rule covers the trip list only.
3. Enter `anonymous` in **Role**, so the rule counts callers who are not signed in.
4. Enter `2` in **Requests per second** and `10` in **Burst**.
5. Keep **Enforce this rule** on and choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/rate-limits \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"endpoint": "GET /api/v1/content/trip", "rate": 2, "burst": 10, "role": "anonymous"}'
```

The answer is `201`.

</div>
</div>

![Opening New rule, choosing the trip route, setting the role, rate and burst, and creating it.](./media/rate-limits-new-rule.webm)

On a free install, **Custom rules** shows a not enabled card in place of the list, and the
protections say that changing them is not enabled on this engine. The global limit and the protections still apply.

![The Limits tab on a free install, with Custom rules locked.](./media/rate-limits-rules-free.png)

![The Limits tab on a licensed install, with the trip rule listed.](./media/rate-limits-rules-licensed.png)

## See who is near a limit

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Open the **Right now** tab. It counts the **Callers at their limit** and lists each caller
close to one, with the **Endpoint**, what is **Left** and when it **Resets in**.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/rate-limits/status \
  -H "Authorization: Bearer $TOKEN"
```

Client addresses come back redacted.

</div>
</div>

The counters are a snapshot from one engine process. With more than one node, each node counts
on its own unless the distributed store is configured.

## Give your tenant its own limit

> **Requires** a license with the `rate-limit-pro` feature to set a limit. Clearing one is
> free.

A tenant limit can only be stricter than the global limit, which still applies.

<div class="paths">
<div data-path="console">

1. Open the **Tenant controls** tab. Under **Tenant limit**, choose **Set a limit**.
2. Enter `20` in **Rate, requests per second** and `40` in **Burst**.
3. Turn on **Enforce this limit** and choose **Save**. **Clear** removes it later.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/rate-limits/tenant-global \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"rate": 20, "burst": 40, "enabled": true}'
```

</div>
</div>

## Block an address

> **Requires** a license with the `rate-limit-pro` feature to add an address. Removing one is
> free.

An allowed address skips your tenant's rules. A denied one is refused with `403` before any
rule is counted, and an address on both lists is allowed.

<div class="paths">
<div data-path="console">

1. On **Tenant controls**, choose **New address**.
2. Pick **Deny** in **List** and enter `203.0.113.0/24` in **Address or range**.
3. Say why in **Note** and choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/rate-limits/ip-rules \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"cidr": "203.0.113.0/24", "list": "deny", "note": "scraper"}'
```

</div>
</div>

Add your own address to the allow list first. A deny entry that covers the address you are
signed in from is refused unless that address is on the allow list.

![Adding a range to the deny list with a note.](./media/rate-limits-deny-address.webm)

On a free install the page says that setting a tenant limit and adding an address are not
enabled on this engine. Saved entries still apply.

![Tenant controls on a free install, with both notices.](./media/rate-limits-tenant-free.png)

![Tenant controls on a licensed install, with a tenant limit and a deny entry.](./media/rate-limits-tenant-licensed.png)

## See what was refused

> **Requires** a license with the `rate-limit-pro` feature.

<div class="paths">
<div data-path="console">

Open the **Refusal history** tab and pick a window, such as **Last hour**. Each row names the
rule or list that refused, the route or address, how many requests it refused and when last.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/rate-limits/history?hours=24" \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

[Rate limiting](../features/rate-limiting.md#what-a-limited-client-sees) shows the answer and
headers a limited client gets.

## Next

[21. AI](ai.md) connects a model provider to your engine. For depth, read
[Rate limiting](../features/rate-limiting.md) and
[API key request limits](../features/api-key-request-limits.md).
