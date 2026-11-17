---
title: "1. Run the engine and the console"
description: "Start the engine, LyEve Console and PostgreSQL on your computer, finish setup, create the first super admin and sign in, in the console or with curl."
sidebar:
  order: 1
covers:
  console:
    - "/"
    - "/setup"
    - "/login"
  api:
    - "GET /api/admin/setup"
    - "GET /api/admin/setup/status"
    - "POST /api/admin/setup"
    - "POST /api/admin/auth/login"
verified:
  engine: v0.62.0
  console: v0.28.0
---

> **Included free** on every install.

In this lesson you start LyEve on your computer, create its first account and sign in. You need
Docker with Compose, `curl`, `openssl` and ports 8080, 3001 and 3002 free. Nothing else is
assumed.

## Start the engine and the console

The stack is the one [Run the engine and the console together](../getting-started/engine-and-console.md)
builds: the engine, the console, PostgreSQL and a small proxy on `http://localhost:8080`. Follow
its steps 1 to 3, with one change. The API steps in this tutorial call the engine's two ports
straight from your shell, so publish them. Add these lines to the `engine` service in
`docker-compose.yml` before you start it:

```yaml
    ports:
      - "3001:3001"
      - "3002:3002"
```

Port 3001 is the Admin API, which configures the instance. Port 3002 is the Content API, which
your sites and apps read. [Core concepts](../concepts/index.md#the-two-apis) explains the split.

When `docker compose ps` shows the engine `(healthy)`, ask whether it waits for setup:

```bash
curl http://localhost:3001/api/admin/setup
```

```json
{"token_source":"env","setup_required":true}
```

`setup_required` stays `true` until the first account exists.

## Tell the engine what it is missing

An engine started with `LYEVE_SETUP_MODE=true` and no database or secrets waits in setup mode
instead of stopping. The Compose file sets everything, so your engine skips this. Read it when
you start an engine on a server before its settings are decided.

<div class="paths">
<div data-path="console">

1. Open `http://localhost:8080`. The setup page says **The engine is missing settings**.
2. Paste the setup token into **Setup token** and choose **Show what to set**.
3. Under **Set these, then restart the engine**, copy the **Environment variables** or the
   **Or in lyeve.yaml** block into the engine's settings. A secret the engine generated is
   shown once, so copy it now.
4. Restart the engine with the command shown under **Restart the engine**. The page waits and
   moves on to creating the first super admin once the engine is configured.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/setup/status \
  -H "X-Setup-Token: <the setup token>"
```

The answer lists each missing setting under `missing`, and carries the lines to paste under
`env` and `yaml`. Set them and restart the engine.

</div>
</div>

![The setup page in setup mode, after the token is entered: the missing settings with a generated secret, the environment lines, the same settings for lyeve.yaml and the restart commands.](./media/run-lyeve-setup-mode.png)

While the console cannot reach the engine, the same page says **Waiting for the engine** and
checks again every few seconds. [Installation](../getting-started/installation.md#setup-mode)
lists what setup mode answers on every route.

## Create the first super admin

Creating the first account takes the setup token, so nobody else who can reach the port claims
the install first. In this stack it is the `SETUP_TOKEN` line of your `.env`:

```bash
grep SETUP_TOKEN .env
```

The password needs at least 12 characters, with an upper-case letter, a lower-case letter and a
digit, and must not be a common password.

<div class="paths">
<div data-path="console">

1. Open `http://localhost:8080`. A new install opens **Set up LyEve**.
2. Paste the token into **Setup token**.
3. Enter your address in **Email**, and the password in **Password** and again in **Confirm
   password**.
4. Choose **Create super admin account**. The console signs you in and opens the **Dashboard**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/setup \
  -H "Content-Type: application/json" \
  -d '{
        "email": "you@example.com",
        "password": "<a password that meets the policy>",
        "setup_token": "<the SETUP_TOKEN value>"
      }'
```

The answer is `201` with the new `user` and a `token`. A second call answers `409`, because an
account now exists.

</div>
</div>

![The setup form is filled in with the token, an email address and the password twice, then submitted, and the console opens the dashboard.](./media/run-lyeve-first-admin.webm)

The account holds the `super_admin` role, which may do everything, in every tenant.

## Sign in

<div class="paths">
<div data-path="console">

1. Open `http://localhost:8080`. Without a session the console opens **Sign in**.
2. Enter your **Email** and **Password** and choose **Sign in**.
3. If your account has a second factor, the form asks for an **Authenticator code**. Enter the
   code from your app and choose **Verify**. Lesson 2 adds one.

</div>
<div data-path="api">

Sign in for a token, and keep it in `TOKEN` for the lessons that follow:

```bash
curl -X POST http://localhost:3001/api/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "<your password>"}'

export TOKEN="<the token from the answer>"
```

The token works on both APIs and lasts 15 minutes by default. When a call answers `401`, sign
in again. An account with a second factor answers `{"mfa_required": true, "challenge_token":
"..."}` instead, which [API endpoints](../reference/api-endpoints.md#authenticate) shows how to
finish.

</div>
</div>

![The sign-in form with email, password, a forgotten password link, a sign-in link by email and the sign-in button.](./media/run-lyeve-sign-in.png)

**Forgot password?** and **Email me a sign-in link** need a way to send email, which lesson 12
sets up. Lesson 2 walks both. A **Sign in with** button appears for each identity provider the
instance is set up for, which lesson 17 adds.

## Run it free or with a license

The `.env` line `LYEVE_LICENSE_KEY=` stays empty for a free install, which is a complete CMS.
Every lesson works on it, and a section that needs a license says so at its top. To try a paid
feature, put a license key on that line and run `docker compose up -d` again, or apply one in
the console in lesson 26. [Licensing and tiers](../concepts/licensing-and-tiers.md) explains
what a key unlocks.

## Next

- [2. Find your way and secure your account](find-your-way.md): read the dashboard and protect
  the account you just made.
- [Installation](../getting-started/installation.md): every setting a production engine needs.
- [Run the engine and the console together](../getting-started/engine-and-console.md): the
  Compose file line by line, and what to do when a step fails.
