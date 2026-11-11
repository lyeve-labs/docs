---
title: Connect the builder
description: "Turn LyEve Builder on for a tenant, approve a builder connection with your password or authenticator code, and end one or all of them."
sidebar:
  order: 2
---

A browser reaches your engine through a builder connection: a key the builder makes in that
browser, approved by a person in your tenant. This page turns the builder on, approves a
connection and ends one.

In LyEve Console, open **Settings** and choose **Builder connections**. An editor, an admin
or a super admin approves and ends connections there, and an admin or a super admin also has
the **Builder settings** tab. With the Admin API, each route takes a session token and nothing
else. An API key, an admin token and a sign-in still waiting for its two-factor code are
refused. [Sign in](../getting-started/quickstart.md) shows how to get a token. The examples
keep it in `TOKEN`.

## Turn the builder on

The builder is off in every tenant until an administrator turns it on.

<div class="paths">
<div data-path="console">

1. Open **Builder connections** and choose the **Builder settings** tab.
2. Turn on **Enabled**. It applies at once, and turning it off asks you to confirm first.
3. Set **Valid for**, **Open to the builder** and **Writes a minute for each connection**, then
   choose **Save**.

</div>
<div data-path="api">

An admin or a super admin reads the settings with `GET /api/admin/builder/settings` and changes
them with `PUT /api/admin/builder/settings`:

```bash
curl -X PUT http://localhost:3001/api/admin/builder/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"allowed": true, "lifetime_days": 90, "show_schemas": ["post"], "writes_per_minute": 120}'
```

</div>
</div>

| Field | Console | What it sets |
|---|---|---|
| `allowed` | **Enabled** | Whether a builder may connect and make calls. Off by default |
| `lifetime_days` | **Valid for** | How long a connection approved from now on lasts: `30`, `90` or `180`. The default is `90`, and connections already approved keep their expiry |
| `show_schemas` | **Open to the builder** | The content types whose fields a page may show, at most 50. An empty list clears it. Each site then chooses from them, as [Show content entries](content.md#what-a-content-type-needs) describes |
| `writes_per_minute` | **Writes a minute for each connection** | How many writes each connection may make in a minute. The default is `120`, and the answer's `max_writes_per_minute` names the highest value you may set |

A field you leave out keeps its value. Turning `allowed` off refuses the next signed call from
every connected browser, while each browser can still check and end its own connection.

## Connect and approve

1. In the builder, enter your engine's address and connect. The builder sends the engine a key
   it made in your browser, and shows you a code, such as `ABCD-EF23`, and the key's
   fingerprint. The code expires after 10 minutes.
2. An editor, an admin or a super admin of the same tenant looks the request up by its code and
   approves it.

<div class="paths">
<div data-path="console">

1. Open **Builder connections** and choose **Approve connection**.
2. Enter the code under **Code** and choose **Find request**.
3. Under **Check this request**, check that **Key** matches the fingerprint the builder shows.
   **From** is sent by the browser and not verified, and a warning says when the request came
   from a different network than your browser. Approve only a code you started yourself.
4. Choose **Valid for**, then enter your **Authentication code**, or **Your password** for an
   account without an authenticator app, and choose **Approve**. **Decline** refuses the
   request.

</div>
<div data-path="api">

1. Look the request up by its code:

   ```bash
   curl http://localhost:3001/api/admin/builder/connections/requests/ABCD-EF23 \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer carries the key's `fingerprint`, the `label` and `origin` the request reported
   about itself, the `lifetime_days` it would get and its `would_expire_at`, and
   `same_network`, which is `false` when the request came from another network than your
   session. Check that `fingerprint` matches the one the builder shows. Approve only a code you
   started yourself.

2. Approve it, naming the fingerprint you checked, the lifetime, and your authenticator code in
   `mfa_code`. An account without an authenticator app sends its `password` instead:

   ```bash
   curl -X POST http://localhost:3001/api/admin/builder/connections/requests/ABCD-EF23/approve \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"fingerprint": "<the fingerprint you checked>", "lifetime_days": 90, "mfa_code": "123456"}'
   ```

   The builder is connected once the answer carries `"status"` and `expires_at`. To refuse the
   request, send `POST .../requests/ABCD-EF23/decline` instead.

</div>
</div>

An account with an authenticator app has to confirm with its current code, and a password alone
is refused for it. A wrong code or password counts toward the account's lockout exactly as a
failed sign-in does. A connection lasts the days you chose, never longer than the tenant allows
and never past the expiry of your own account. It can do no more than you can: a role that may
not change a page cannot have the builder change it either.

An unknown code, a code from another tenant, an expired code and a request someone already
decided all answer the same `404`. One person may look up 10 codes a minute, and up to 10
requests may wait from one address or IPv6 `/64`, and 50 in one tenant.

## See and end connections

<div class="paths">
<div data-path="console">

**Builder connections** lists the connections ever approved, with **Approved by**, **Key**,
**Approved on**, **Last used**, **Expires** and **Status**: every one of the tenant's for an
admin, and their own for an editor. **Revoke** ends one connection. An admin's **Revoke all**
ends every connection of the tenant and declines every waiting request.

</div>
<div data-path="api">

| Method | Path | Who | Does |
|---|---|---|---|
| `GET` | `/api/admin/builder/connections` | An editor and up | Lists the connections ever approved: every one of the tenant's for an admin, and their own for an editor |
| `POST` | `/api/admin/builder/connections/{id}/revoke` | An editor and up | Ends one connection: any of the tenant's for an admin, and one they approved for an editor |
| `POST` | `/api/admin/builder/connections/revoke-all` | An admin and up | Ends every connection of the tenant and declines every waiting request |

</div>
</div>

A connection that ended answers no signed call, so the builder in that browser has to connect
again. Waiting and declined requests are deleted an hour after their code expires, and ended
connections 90 days after they end.

## Related

- [Sites and pages](sites-and-pages.md): what the builder makes once it is connected.
- [Multi-factor authentication](../features/multi-factor-auth.md): turn on the authenticator
  app an approval asks for.
