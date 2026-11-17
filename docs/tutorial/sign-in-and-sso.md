---
title: "17. Sign-in, SSO and provisioning"
description: "Check which security controls protect your engine, let people sign in with OpenID Connect or SAML, provision accounts from a directory with SCIM, and set the sign-in window and the captcha."
sidebar:
  order: 17
covers:
  console:
    - "/admin/settings/security"
    - "/admin/settings/oauth"
    - "/admin/settings/saml"
    - "/admin/settings/scim"
    - "/admin/settings/sign-in"
    - "/admin/settings/captcha"
  api:
    - "GET /api/admin/security/controls"
    - "POST /api/admin/oauth-templates/apply"
    - "POST /api/admin/saml-providers"
    - "POST /api/admin/scim/providers"
    - "PUT /api/admin/auth/provider-window"
    - "PUT /api/admin/captcha/settings"
verified:
  engine: v0.62.0
  console: v0.28.0
---

The accounts you made in [lesson 16](users-and-roles.md) sign in with a password. In this
lesson you check how the engine is protected, then let people sign in with the account they
already have at Google, Okta or Microsoft, and let a directory create and remove accounts on
its own. Sign in as a super admin, and keep a session token in `TOKEN`.

## Check which controls are on

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Open **Settings > Security**. The top row counts the controls that are enforcing and the ones
that are not. Each control below has a card with its state and what it protects against. A
card that is not enforcing says **To turn it on:** and what to do.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/security/controls \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

When a control is switched on and not in the request path, the page says so at the top. Fix
that first, because the engine looks protected and is not.

![The Security page with its counts and a card that says how to turn a control on.](./media/sign-in-and-sso-security.png)

## Sign in with Google

> **Included free** on every install, with one sign-in provider. More providers, the Okta and
> Azure AD templates and role mapping need a license with the `oauth-pro` feature.

Set `LYEVE_BASE_URL` on the engine to its public address first, because every callback
address is built from it. Then create an OAuth client at Google and register
`<LYEVE_BASE_URL>/api/admin/auth/oauth/google/callback` as its redirect address.

<div class="paths">
<div data-path="console">

1. Open **Access > OAuth providers** and choose **New provider**.
2. Enter `google` in **Provider name** and `https://accounts.google.com` in **Issuer URL**.
3. Enter your client's **Client ID** and **Client secret**. Leave **Scopes**,
   **Roles claim** and **Default roles** empty to keep their defaults.
4. Turn **Enabled** on and choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/oauth-templates/apply \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key": "google", "overrides": {"client_id": "1234.apps.googleusercontent.com", "client_secret": "<client secret>", "enabled": true}}'
```

</div>
</div>

![Opening New provider, filling in Google's issuer and client, and enabling it.](./media/sign-in-and-sso-oauth.mp4)

The sign-in page now offers a button for Google. [OAuth sign-in](../features/oauth-sign-in.md)
covers role mapping and what happens when an email already has an account.

## Sign in through a SAML provider

> **Requires** a license with the `saml` feature.

<div class="paths">
<div data-path="console">

1. Open **Access > SAML SSO** and choose **New provider**.
2. Pick your provider in **Start from**, which fills in the usual settings.
3. Enter `okta` in **Name**. It is the label on the sign-in button.
4. Paste the provider's identifier into **Entity ID**, its single sign-on URL into
   **Sign-on URL** and its certificate into **Their signing certificate**.
5. Leave **Show on the sign-in page** on and choose **Create**.

The row links the **Metadata** you give the provider, and **Test sign-in** tries it.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/saml-providers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "okta",
        "entity_id": "http://www.okta.com/exk1a2b3c4d5e6f7g8h9",
        "sso_url": "https://example.okta.com/app/example_lyeve/exk1a2b3c4d5e6f7g8h9/sso/saml",
        "idp_cert": "-----BEGIN CERTIFICATE-----\nMIID...\n-----END CERTIFICATE-----",
        "attributes_mapping": {"email": "email", "roles": "groups"},
        "enabled": true
      }'
```

</div>
</div>

A new provider starts with **Require signed assertions**, **Require a signed response** and
**Sign requests to the provider** on. When a certificate nears its end, **Certificates to
renew** counts it, and **Prepare a new certificate** on the row starts the switch.

On an install without the feature, **SAML SSO** shows a card that says it is not enabled on
this engine.

![The SAML SSO page on a free install, with the not enabled card.](./media/sign-in-and-sso-saml-free.png)

![The SAML SSO page on a licensed install, with the okta provider and its certificates.](./media/sign-in-and-sso-saml-licensed.png)

## Let a directory manage accounts

> **Requires** a license with the `scim` feature.

SCIM lets a directory such as Microsoft Entra or Okta create, update and remove accounts here
without anyone signing in. Make a long random token first:

```bash
SCIM_TOKEN=$(openssl rand -hex 32)
echo "$SCIM_TOKEN"
```

<div class="paths">
<div data-path="console">

1. Open **Access > SCIM provisioning** and choose **New connection**.
2. Enter `entra` in **Name** and paste the token into **Bearer token**. It is stored as a
   hash, so this is the only time it can be read.
3. Under **When the directory removes somebody**, choose what happens to the account, then
   choose **Create**.

**Where the directory sends its requests** gives the **Base path** the directory calls, with
the token as its only credential.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/scim/providers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "entra", "bearer_token": "'"$SCIM_TOKEN"'"}'
```

The answer is `201`, and the only time the token comes back.

</div>
</div>

![Opening New connection, pasting the token, choosing what a removal does and creating it.](./media/sign-in-and-sso-scim.mp4)

**Rotate token**, on a connection's row, replaces the token. Provisioning fails until the
directory holds the new one.

## Close the other ways in

> **Included free** on every install.

An account your identity provider created can also sign in with a password, reset it or ask
for an email link, but only within a number of days of its last sign-in through the provider.

<div class="paths">
<div data-path="console">

Open **Settings > Sign-in**. Under **Without the identity provider**, enter the number in
**Days since the last provider sign-in** and choose **Save**. `0` allows sign-in through the
provider only.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/auth/provider-window \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"days": 7}'
```

</div>
</div>

Removing someone at your provider tells the engine nothing, so the other ways close when the
window passes. With SCIM they close at once.

## Set the tenant's captcha

> **Included free** on every install, with one captcha for the instance. A provider and keys
> of a tenant's own need a license with the `multitenant-customization` feature.

The instance captcha is set in the engine's environment, as
[Captcha](../features/captcha.md#turn-it-on) shows. This screen sets the one your tenant's own
pages and flows check. Signing in to the console always uses the instance captcha, so a wrong
key here cannot lock you out.

<div class="paths">
<div data-path="console">

1. Open **Settings > Captcha**. **What is checked now** names the captcha in use.
2. Under **Tenant settings**, pick a **Provider** and enter the **Site key** and the
   **Secret key**.
3. Choose **Save**. **Clear tenant settings** puts the tenant back on the instance captcha.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/captcha/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"provider": "hcaptcha", "site_key": "<site key>", "secret_key": "<secret key>"}'
```

</div>
</div>

On a free install, **Tenant settings** says tenant captcha settings are not enabled on this
engine, and settings already saved still apply.

![The Captcha page on a free install, with the tenant settings locked.](./media/sign-in-and-sso-captcha-free.png)

![The Captcha page on a licensed install, with a tenant provider and its site key.](./media/sign-in-and-sso-captcha-licensed.png)

## Next

[18. Privacy, compliance and the audit log](privacy-and-audit.md) answers a person who asks
for their data. For depth, read [Sign-in options](../integrations/oauth-sso.md),
[OAuth sign-in](../features/oauth-sign-in.md), [SAML single sign-on](../features/saml-sso.md),
[SCIM provisioning](../features/scim-provisioning.md), [Captcha](../features/captcha.md) and
[Security controls](../trust/security-controls.md).
