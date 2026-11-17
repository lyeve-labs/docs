---
title: "2. Find your way and secure your account"
description: "Read the dashboard and the sidebar, choose how the console looks, add an authenticator app, trust a device, approve a command-line sign-in and recover a lost password."
sidebar:
  order: 2
covers:
  console:
    - "/admin"
    - "/admin/settings/account"
    - "/admin/settings/mfa"
    - "/admin/settings/devices"
    - "/admin/device"
    - "/login/magic-link"
    - "/auth/magic-link/verify"
    - "/forgot-password"
    - "/reset-password"
  api:
    - "GET /api/admin/auth/me"
    - "GET /api/admin/mfa/status"
    - "POST /api/admin/mfa/totp/setup"
    - "POST /api/admin/mfa/totp/verify"
    - "POST /api/admin/mfa/totp/disable"
    - "GET /api/admin/devices"
    - "POST /api/admin/devices/trust"
    - "POST /api/admin/devices/{id}/untrust"
    - "POST /api/admin/auth/device"
    - "GET /api/admin/auth/device/{user_code}"
    - "POST /api/admin/auth/device/{user_code}/approve"
    - "POST /api/admin/auth/device/{user_code}/deny"
    - "POST /api/admin/auth/magic-link/request"
    - "POST /api/admin/auth/magic-link/verify"
    - "POST /api/admin/auth/password-reset/request"
    - "POST /api/admin/auth/password-reset/confirm"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson you learn where things are in LyEve Console, then protect the account you
created. You need the stack and the super admin from [lesson 1](run-lyeve.md), and a token in
`TOKEN` for the API steps.

## Read the dashboard

> **Included free** on every install.

The **Dashboard** opens after you sign in. It is a console screen with no single route behind
it: each panel reads the feature it reports on.

![The dashboard of a new install: the window choice, what needs attention, the counts under Right now and the recent changes.](./media/find-your-way-dashboard.png)

| Part | What it does |
|---|---|
| **Window** | Sets the period every panel covers: **6 hours**, **24 hours** or **7 days** |
| **Needs attention** | Lists what is failing or about to, each with a link to the screen that fixes it. It says so when nothing does |
| **Right now** | Counts **Content types**, **Entries**, **Failed jobs** and **Webhooks failing**, and shows the **License** |
| **Traffic** | **Requests per hour** and the **Busiest endpoints**, with their error rate |
| **What changed** | **Recent entries**, **Recent audit entries**, **Dead-lettered deliveries** and **Recent errors** |

A panel appears only while the feature it reads runs on your install, and only for a role that
may read it. On a new install with no content types, the dashboard offers **New content type**,
which lesson 3 uses. An administrator can change the layout with **Edit dashboard**, as lesson
26 shows.

The sidebar holds the daily work at the top (**Content**, **Schema builder**, **Flows**,
**Reviews**, **Media library** and **Search**), then the groups **Delivery**, **Access**,
**Operations**, **Insight** and **Settings**. [LyEve Console](../console/index.md#finding-your-way)
says what each group holds and which role sees it.

## See your account and choose a color scheme

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Open the account menu in the header and choose **Account and appearance**.
2. **Account** shows your address and roles, with links to **Two-factor authentication** and
   **Trusted devices**.
3. Under **Appearance**, pick a **Color scheme**. **System** follows your desktop setting. The
   choice holds for this browser only.

</div>
<div data-path="api">

The API holds no appearance setting, because the choice lives in the browser. Read who you are
signed in as:

```bash
curl http://localhost:3001/api/admin/auth/me \
  -H "Authorization: Bearer $TOKEN"
```

The answer carries your `id`, `email` and `roles`.

</div>
</div>

## Turn on two-factor authentication

> **Included free** on every install. Enrolling a passkey needs a license with the `mfa-pro`
> feature.

<div class="paths">
<div data-path="console">

1. Open the account menu and choose **Two-factor authentication**. The page says
   **Two-factor authentication is off for this account.**
2. Choose **Turn on two-factor authentication**. If you signed in a while ago, the page first
   asks for your **Password**.
3. Scan the QR code with an authenticator app, or type the **Manual entry secret** into it.
4. Enter **The 6-digit code from the app** and choose **Verify and turn on**.
5. Save the backup codes the page shows, or choose **Copy all codes**, then choose **Done**.
   They are not shown again.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/mfa/status \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/mfa/totp/setup \
  -H "Authorization: Bearer $TOKEN"
```

The second answer carries a `secret` and a `provisioning_uri`. Add the secret to your app, then
confirm with the code it shows:

```bash
curl -X POST http://localhost:3001/api/admin/mfa/totp/verify \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"code": "123456"}'
```

The answer is `{"enabled": true, "backup_codes": [...]}`. Save the codes.

</div>
</div>

![Two-factor authentication is turned on: the password check, the QR code, the code typed from the app, and the backup codes with Copy all codes.](./media/find-your-way-mfa-on.mp4)

From the next sign-in, the console asks for an **Authenticator code** after your password. To
move to another phone, choose **Replace the authenticator app**. To stop, choose **Turn off the
authenticator app**, which takes a current code or a backup code, or `POST /api/admin/mfa/totp/disable` with
`{"code": "..."}`. [Multi-factor authentication](../features/multi-factor-auth.md) covers
passkeys, backup codes and resetting someone else's factor.

## Trust this device

> **Requires** a license with the `device-fingerprint` feature.

A trusted device signs in without the extra check a risky sign-in gets.

<div class="paths">
<div data-path="console">

1. Open the account menu and choose **Trusted devices**.
2. Under **This device**, enter a name in **Name this device**, such as `Work laptop`, and
   choose **Trust this device**.
3. The device appears under **Your trusted devices** with when it was **Last seen** and
   **Added**. To forget one, use its button under **Actions**.

Each browser counts as its own device, so open the page in each browser you want to trust.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/devices/trust \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"label": "Work laptop"}'

curl http://localhost:3001/api/admin/devices \
  -H "Authorization: Bearer $TOKEN"
```

Forget a device with `POST /api/admin/devices/{id}/untrust`, using the `id` from the list.

</div>
</div>

On a free install, the page says the feature is not enabled on this engine and links to how to
enable it. With the license, it shows the form and the list.

![Trusted devices on a free install: a notice that the feature is not enabled on this engine.](./media/find-your-way-devices-free.png)

![Trusted devices with a license: this browser is named Work laptop and trusted, and it appears in the list.](./media/find-your-way-devices-licensed.mp4)

[Trusted devices](../features/trusted-devices.md) explains the risk scoring, brute-force blocks
and login history.

## Approve a sign-in from a command-line tool

> **Included free** on every install.

A tool on a machine without a browser signs in by showing you a short code. You approve the
code in the console, and the tool receives a session with your roles.

<div class="paths">
<div data-path="console">

1. The tool shows a code and a link to the console's `/admin/device` page. Open the link while
   signed in.
2. On **Approve a device sign-in**, check that **Code from the device** matches the code the
   tool shows.
3. Enter **Your password**, or an **Authentication code** from your app once two-factor is on.
4. Choose **Approve**. Choose **Deny** if you did not start this sign-in yourself.

</div>
<div data-path="api">

The tool starts the sign-in, with no credential:

```bash
curl -X POST http://localhost:3001/api/admin/auth/device
```

The answer carries a `device_code` for the tool and a `user_code` for you. Look the request
up, then approve it with your password, or with `mfa_code` in place of `password`:

```bash
curl http://localhost:3001/api/admin/auth/device/<user_code> \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/auth/device/<user_code>/approve \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"password": "<your password>"}'
```

`POST /api/admin/auth/device/<user_code>/deny` refuses it.

</div>
</div>

![A device sign-in is approved: the code from the device matches the terminal, the password is entered and Approve is chosen.](./media/find-your-way-device-approve.mp4)

The engine builds the link from `LYEVE_CONSOLE_URL`, which the Compose file sets.
[API endpoints](../reference/api-endpoints.md) lists the polling route the tool calls.

## Sign in with an email link

> **Included free** on every install.

Magic link sign-in sends a single-use link instead of asking for a password. It needs a way to
send email. Until lesson 12 sets one up, the request answers the same and no mail leaves.

<div class="paths">
<div data-path="console">

1. On **Sign in**, choose **Email me a sign-in link**.
2. Enter your **Email** and choose **Email me a link**. The page says a link is on its way if
   an account exists for that address.
3. Open the link from the email. On **Sign in with your link**, choose **Continue**. An account
   with a second factor is asked for its **Authenticator code** on the same page.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/auth/magic-link/request \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

Redeem the `token` from the emailed link:

```bash
curl -X POST http://localhost:3001/api/admin/auth/magic-link/verify \
  -H "Content-Type: application/json" \
  -d '{"token": "<token from the link>"}'
```

</div>
</div>

![A sign-in link is requested, then opened from the email, and Continue signs in.](./media/find-your-way-magic-link.mp4)

[Magic link sign-in](../features/magic-link-sign-in.md) covers sign-up by link and the rate
limits.

## Reset a forgotten password

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. On **Sign in**, choose **Forgot password?**.
2. Enter your **Email** and choose **Email me a reset link**. The link works once and expires
   in an hour.
3. Open the link. On **Choose a new password**, enter it in **New password** and **Confirm new
   password**. **Remove my passkeys** is ticked, which is the safe choice.
4. Choose **Save**, then **Sign in** with the new password.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/auth/password-reset/request \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'

curl -X POST http://localhost:3001/api/admin/auth/password-reset/confirm \
  -H "Content-Type: application/json" \
  -d '{"token": "<token from the link>", "password": "<a new password>"}'
```

</div>
</div>

![A password reset is requested, the link from the email opens Choose a new password, the new password is saved, and the page offers Sign in.](./media/find-your-way-reset-password.mp4)

Every session the account held ends when the password changes.
[Password reset](../features/password-reset.md) has the limits and what to check when no mail
arrives.

## Next

- [3. Model your content](model-content.md): create the content types of the travel guide.
- [LyEve Console](../console/index.md): the sidebar and what each role sees.
- [Secure your instance](../how-to/secure-your-instance.md): the settings a public instance needs.
