---
title: "15. Connected editors"
description: "Turn the builder on for your tenant, open trips to it, approve a browser that edits your pages, and end a connection."
sidebar:
  order: 15
covers:
  console:
    - "/admin/settings/connected-editors"
    - "/admin/settings/connected-editors/builder"
  api:
    - "PUT /api/admin/builder/settings"
    - "GET /api/admin/builder/connections/requests/{code}"
    - "POST /api/admin/builder/connections/requests/{code}/approve"
    - "GET /api/admin/builder/connections"
    - "POST /api/admin/builder/connections/{id}/revoke"
verified:
  engine: v0.62.0
  console: v0.28.0
---

> **Included free** on every install.

The builder designs your site's pages in the browser and stores them in your engine. A
browser reaches the engine through a connection that a person in your tenant approves. In
this lesson you turn the builder on, open `trip` to it, and approve your own browser. You need
the `trip` content type from lesson 3 and a session token in `TOKEN`, as in
[lesson 14](experiments-and-analytics.md). These routes refuse an API key and an admin token.

In the console, open **Settings** and choose **Builder connections**. Your account menu has
the same link.

## Turn the builder on and open trips to it

The builder is off in every tenant until an `admin` or a `super_admin` turns it on.

<div class="paths">
<div data-path="console">

1. On **Builder connections**, choose the **Builder settings** tab.
2. Turn on **Enabled**. It applies at once.
3. Pick how long a connection lasts in **Valid for**.
4. Under **Open to the builder**, choose `trip`, so pages can show trips.
5. Keep **Writes a minute for each connection** and choose **Save**.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/builder/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"allowed": true, "lifetime_days": 90, "show_schemas": ["trip"], "writes_per_minute": 120}'
```

</div>
</div>

![The Builder settings tab with Enabled on, a lifetime chosen and trip open to the builder.](./media/connected-editors-settings.png)

The tab also holds **Where forms can send**, which lets a form on a page save posts into a
content type, and **Flows**, which lets a form start a flow.
[Forms](../builder/forms.md) walks through both.

## Approve your browser

1. In the builder, enter your engine's address and connect. The builder shows a code, such as
   `ABCD-EF23`, and the fingerprint of the key it made in your browser. The code expires after
   10 minutes.
2. Approve the code in your tenant.

<div class="paths">
<div data-path="console">

1. On **Builder connections**, choose **Approve connection**.
2. Enter the code under **Code** and choose **Find request**.
3. Under **Check this request**, check that **Key** matches the fingerprint the builder shows.
   A warning says when the request came from a different network than your browser.
4. Choose **Valid for**, confirm with your authenticator code or your password, and choose
   **Approve**. **Decline** refuses the request.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/builder/connections/requests/ABCD-EF23 \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/builder/connections/requests/ABCD-EF23/approve \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"fingerprint": "<the fingerprint you checked>", "lifetime_days": 90, "mfa_code": "123456"}'
```

An account without an authenticator app sends `password` in place of `mfa_code`.

</div>
</div>

![Entering the code from the builder, checking the key, confirming and approving.](./media/connected-editors-approve.webm)

Approve only a code you started yourself. If someone sends you a code, they are asking for
access to your pages. A connection can do no more than you can.

## See and end connections

<div class="paths">
<div data-path="console">

The list shows each connection with **Approved by**, **Key**, **Approved on**, **Last used**,
**Expires** and **Status**. An admin sees every connection of the tenant, and an editor sees
their own. **Revoke** ends one. **Revoke all**, for an admin, ends every connection and
declines every waiting request.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/builder/connections \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/builder/connections/$CONNECTION_ID/revoke \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

A connection that ended answers no signed call, so the builder in that browser has to connect
again.

## Next

[16. Users, roles and permissions](users-and-roles.md) adds the people who work in your
tenant. For depth, read [Connect the builder](../builder/connect.md),
[Show content entries](../builder/content.md) and [Builder](../features/builder.md).
