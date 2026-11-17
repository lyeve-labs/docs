---
title: "26. Customize the console"
description: "Find a setting, read where each engine setting comes from and change one, apply a license, then give a tenant's console its own name, menu, dashboard and pages."
sidebar:
  order: 26
covers:
  console:
    - "/admin/settings"
    - "/admin/settings/configuration"
    - "/admin/settings/license"
    - "/admin/settings/customization"
    - "/admin/settings/customization/menu"
    - "/admin/settings/customization/dashboard"
    - "/admin/settings/customization/pages"
    - "/admin/settings/customization/pages/[slug]"
    - "/admin/pages/[slug]"
  api:
    - "GET /api/admin/config"
    - "PUT /api/admin/config"
    - "POST /api/admin/license/renew"
    - "GET /api/admin/customization"
    - "PUT /api/admin/customization"
    - "PUT /api/admin/customization/dashboard"
    - "PUT /api/admin/customization/pages/{slug}"
verified:
  engine: v0.62.0
  console: v0.28.0
---

This is the last lesson in the console. You find the settings, change one of the engine's own,
apply a license, and then make the travel team's console their own: its name and color, a
shorter menu, a dashboard of trips and a page of their own. You need a super admin account and
a token in `TOKEN` for the API steps. [Lesson 25](captures-and-telemetry.md) is the one before
this.

## Find a setting

> **Included free** on every install.

Open **Settings** in the sidebar. The page groups every settings screen by what you came to
change: **Your account**, **Tenant**, **Sign-in and access**, **APIs and delivery**,
**Data and compliance** and **Engine**. Each entry says what it sets. A screen appears only
while its feature runs and your role may open it. This page is in the console only.

![The Settings page, with the settings screens grouped by what they change.](./media/customize-the-console-settings.png)

## Read and change an engine setting

> **Included free** on every install.

The engine reads each setting from an environment variable first, then its YAML file, then
the value saved in the console. Here you keep finished flow runs for a week.

<div class="paths">
<div data-path="console">

1. Open **Settings > Configuration**. **Where the values come from** counts the settings each
   place sets, and **How to set a setting** explains the order.
2. Type `flow_run_retention` in **Filter settings**. The row shows the value, its source and
   its default.
3. Enter `168h` in the row and choose **Save**. The row says whether the value applies now or
   at the next restart.

A value an environment variable or the file sets is read-only here, because that place wins.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/config \
  -H "Authorization: Bearer $TOKEN"

curl -X PUT http://localhost:3001/api/admin/config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"values": {"flow_run_retention": "168h"}}'
```

The answer lists each saved setting under `saved`, and one a higher place holds under
`refused`.

</div>
</div>

![Filtering Configuration for flow_run_retention, entering a new value and saving it.](./media/customize-the-console-configuration.webm)

[Change a setting](../how-to/change-a-setting.md) covers the environment and the file, and
[Configuration](../reference/configuration.md) lists every setting.

## Apply a license

> **Included free** on every install.

A license turns on the paid features and capabilities it names. A free install needs none.

<div class="paths">
<div data-path="console">

1. Open **Settings > License**. **Status** shows the plan, its state and the expiry.
2. Paste the token or key you were issued into **License token or key**.
3. Choose **Activate license**, or **Renew license** when one is already in force. The page says
   **License applied. Plugins are reloading.**

</div>
<div data-path="api">

The route takes a token from signing in, not an admin token:

```bash
curl -X POST http://localhost:3001/api/admin/license/renew \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"license_key": "<token or key>"}'
```

</div>
</div>

[Licensing and tiers](../concepts/licensing-and-tiers.md#apply-a-license) says what applies at
once and what waits for a restart.

## Give the console its own name and color

> **Requires** a license with the `multitenant-customization` feature. See [pricing](https://lyeve.com/pricing/).

Everything on **Customization** is kept per tenant and applies to everyone in it.

<div class="paths">
<div data-path="console">

1. Open **Settings > Customization**. The **Brand** tab opens.
2. Enter `Trips desk` in **Name**, `#0F766E` in **Accent color** and
   `Publish new trips before noon on Fridays.` in **Welcome line**.
3. Choose **Save**. The sidebar and the tab title read **Trips desk**.

Under **Accent color**, a preview shows the color in the dark and the light theme with its
contrast. **Logo** and **Tab icon** below each take an image with a **Save** of their own.

</div>
<div data-path="api">

`PUT` replaces the brand and the menu together, so read them first and send them back whole:

```bash
curl http://localhost:3001/api/admin/customization \
  -H "Authorization: Bearer $TOKEN"

curl -X PUT http://localhost:3001/api/admin/customization \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"brand": {"name": "Trips desk", "accent": "#0F766E", "welcome": "Publish new trips before noon on Fridays."}, "menu": {"hidden": [], "pinned": [], "links": []}}'
```

</div>
</div>

On a free install every **Customization** tab shows a lock and **Not enabled on this engine**,
and the console keeps its stock name, menu and dashboard.

![The Customization page on a free install, with the not enabled card.](./media/customize-the-console-brand-free.png)

![The Brand tab on a licensed install, with the name, accent color, welcome line and the theme preview.](./media/customize-the-console-brand-licensed.png)

## Shorten the menu

> **Requires** a license with the `multitenant-customization` feature. See [pricing](https://lyeve.com/pricing/).

<div class="paths">
<div data-path="console">

1. On **Customization**, choose the **Menu** tab.
2. In **Hidden**, pick the entries the team never uses. Hiding removes the entry from the menu
   only, and the page keeps its own access rules.
3. In **Pinned**, pick **Content** and **Media library**. They move to a group of their own at
   the top.
4. Choose **Save**. The page says **Menu saved.**
5. Under **Links**, choose **New link**. Enter `Style guide` in **Label** and the address of
   your team's style guide in **Address**, leave **Seen by** empty for everyone, and choose
   **Create**.

</div>
<div data-path="api">

The menu travels with the brand. Send both, with the menu filled in:

```bash
curl -X PUT http://localhost:3001/api/admin/customization \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"brand": {"name": "Trips desk", "accent": "#0F766E", "welcome": "Publish new trips before noon on Fridays."}, "menu": {"hidden": [], "pinned": [], "links": [{"label": "Style guide", "url": "https://example.com/style-guide"}]}}'
```

</div>
</div>

## Build the dashboard

> **Requires** a license with the `multitenant-customization` feature. See [pricing](https://lyeve.com/pricing/).

<div class="paths">
<div data-path="console">

1. On **Customization**, choose the **Dashboard** tab.
2. Under **Add a widget**, choose **Add entry counts**. Pick `trip` and `trip_stop` in
   **Content types**, and enter `Trips` in **Heading**.
3. Choose **Add latest entries**, pick `trip` in **Content type** and set **Entries** to `5`.
4. Use the arrows on a widget to change its place, **Width** to set how much of the row it
   takes, and **Seen by** to limit it to some roles.
5. Under **Preview**, pick a role and choose **Preview** to see the layout as that role would.
6. Choose **Save**. Everyone in this tenant now sees the widgets their role allows.

**Reset to the stock dashboard** brings back the dashboard every install starts with.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/customization/dashboard \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"widgets": [{"id": "trips", "type": "entry_counts", "width": "half", "title": "Trips", "schemas": ["trip", "trip_stop"]}, {"id": "latest", "type": "latest_entries", "width": "half", "schema": "trip", "limit": 5}]}'
```

</div>
</div>

![Adding an entry counts widget for trip and trip_stop and a latest entries widget, previewing as an editor and saving.](./media/customize-the-console-dashboard.webm)

A widget shows only what its viewer could open anyway.
[Page blocks and dashboard widgets](../features/tenants.md#brand-the-console-per-tenant) lists
every widget.

## Add a page of your own

> **Requires** a license with the `multitenant-customization` feature. See [pricing](https://lyeve.com/pricing/).

A page gets its own menu entry and opens at `/admin/pages/` followed by its address.

<div class="paths">
<div data-path="console">

1. On **Customization**, choose the **Pages** tab, then **New page**.
2. Enter `This week's trips` in **Title** and `this-week` in **Address**, and choose **Create**.
   The page's editor opens.
3. Choose **Add text** and write the week's notes in **Text**, in Markdown.
4. Choose **Add content list**, pick `trip` in **Content type** and **Published** in **Status**.
5. Choose **Save**, then **View page**. The page opens with its blocks, and the menu holds its
   entry.

On the **Dashboard** tab, **Page above the dashboard** shows one of these pages at the top of
the dashboard.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/customization/pages/this-week \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"slug": "this-week", "title": "This week'"'"'s trips", "description": "", "roles": [], "blocks": [{"type": "markdown", "body": "New trips go out on Friday."}, {"type": "content", "schema": "trip", "status": "published", "limit": 10}]}'
```

</div>
</div>

![Creating a page, adding a text block and a list of published trips, saving it and viewing it.](./media/customize-the-console-page.webm)

## Next

The next four lessons leave the console and work on the engine itself:
[27. Configure and deploy the engine](configure-and-deploy.md). For depth, read
[Change a setting](../how-to/change-a-setting.md),
[Licensing and tiers](../concepts/licensing-and-tiers.md) and
[Brand the console per tenant](../features/tenants.md#brand-the-console-per-tenant).
