---
title: Sign-in options
description: Compare every way people can sign in to the admin console, from passwords and magic links to OAuth, SAML, SCIM, MFA and trusted devices, and pick one.
sidebar:
  order: 7
---

> **Requires** a license with the `saml`, `scim` or `device-fingerprint` feature for those options. Passwords, magic links, OAuth sign-in, MFA and captcha are free, and OAuth sign-in and MFA have paid parts named in the tables. See [pricing](https://lyeve.com/pricing/).

Every way of signing in ends the same way: the person holds a LyEve session
token, the same one a password sign-in gets, and your services verify it the
same way (see [validate tokens](validate-tokens-jwks.md)). What differs is who
checks the person's identity, and what extra checks stand in front of it. Most
instances combine several options: a way in, a second factor, and something
that slows down attackers.

## Ways in

| Option | Best for | Free or license | Page |
|---|---|---|---|
| Password | Every instance. The first super admin always signs in this way. | Free | [Quickstart](../getting-started/quickstart.md), and [password reset](../features/password-reset.md) for a forgotten one |
| Magic link | Occasional editors who would rather click an emailed link than keep a password. | Free | [Magic link sign-in](../features/magic-link-sign-in.md) |
| OAuth and OpenID Connect | Teams on Google Workspace, Okta, Azure AD, Keycloak or any OIDC provider. | Free with one provider. More providers, the Okta and Azure AD templates and role mapping need `oauth-pro` | [OAuth sign-in](../features/oauth-sign-in.md) |
| SAML | Companies whose identity provider speaks SAML 2.0, with a provider per tenant. | Requires `saml` | [SAML single sign-on](../features/saml-sso.md) |

## Accounts from your directory

| Option | Best for | Free or license | Page |
|---|---|---|---|
| SCIM provisioning | Creating, updating and removing accounts from Azure AD, Okta or another directory before anyone signs in. Pair it with SAML. | Requires `scim` | [SCIM provisioning](../features/scim-provisioning.md) |

## Accounts a tenant's provider creates

An account that SAML, OAuth or SCIM creates, or that a magic link sign-up
creates, belongs to the tenant it was created in. LyEve finds it by that tenant
and its address. On an install with several tenants, the same address can be
a different account in each tenant, and a sign-up, password reset or magic link
answers the same whether another tenant holds the address or nobody does. An
account that signs in with a password under its own address works as before.
The console's user list marks these accounts **Signs in through your identity
provider**.

Because an address can repeat across tenants, a session's `email` claim does
not identify an account on its own. Identify an account by its id, the `sub`
claim. See [validate tokens](validate-tokens-jwks.md#what-a-token-holds).

### Name the organization

Password sign-in, password reset and magic link take an optional `tenant`
field in the request body. The tenant the request's hostname resolves always
wins over it, so the field counts only where the hostname names no tenant. The
console's sign-in, forgot password and email link pages show it as
**Organization (optional)**. A reset link carries the account's tenant, and
the sign-in page it leads to fills the field in.

### Sign-in without the provider

An account a provider signs in can also sign in with a password, reset its
password or ask for a magic link. Each tenant decides how many days after the
account's last sign-in through its provider those ways stay open. After that,
they answer as they do for an unknown address or a wrong password.

| Setting | Values | Default |
|---|---|---|
| Days since the last provider sign-in | `0` to `365`. `0` allows sign-in through the provider only. | `30` |

Removing someone at your provider tells LyEve nothing, so the other ways close
when the window passes. To close them at once, provision with
[SCIM](../features/scim-provisioning.md). A person the directory deactivates
is refused at SAML sign-in, password reset and magic link from that moment,
whatever the window says. An account that a magic link sign-up created has no
provider, so the window does not apply to it.

A tenant admin sets the window:

<div class="paths">
<div data-path="console">

Open **Settings > Sign-in**. Under **Without the identity provider**, enter
the number in **Days since the last provider sign-in** and choose **Save**.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/auth/provider-window \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"days": 7}'
```

```json
{ "days": 7 }
```

</div>
</div>

`GET /api/admin/auth/provider-window` answers the same shape. Both routes need
the `admin` or `super_admin` role and a signed-in session. An API key or an
admin token is refused with `403`. An admin acts on their own tenant, and a
super admin on the tenant named with `X-Tenant-ID`. A value that is not a whole
number from 0 to 365 answers `422`. A change applies to the next sign-in, and
each one is written to the [audit log](../features/audit-log.md) as
`auth.provider_window.update` with the old and new values.

When you upgrade to 0.55.0, accounts that OAuth created and accounts that an
active SCIM user links count as signed in through their provider at the
upgrade, so they keep their other ways in for one window. An account that
SAML created without SCIM starts its window at its next SAML sign-in. No
window applies to it until then, so its password, password reset and magic
link stay open.

## Extra checks at sign-in

| Option | Best for | Free or license | Page |
|---|---|---|---|
| Multi-factor auth and passkeys | A second step after the password, or a passkey with no password at all. | Free. Enrolling a passkey needs `mfa-pro` | [Multi-factor authentication](../features/multi-factor-auth.md) |
| Trusted devices | Asking for the second factor only when a sign-in looks risky, and blocking brute force on every sign-in route. | Requires `device-fingerprint` | [Trusted devices](../features/trusted-devices.md) |
| Captcha | A challenge after a few failed password sign-ins. | Free | [Captcha](../features/captcha.md) |

Every install also allows 5 password sign-in attempts per address every 15
minutes, and refuses an account's password sign-in after 5 failures within 15
minutes. See [rate limiting](../features/rate-limiting.md).

## Choose

- **A small team with no identity provider:** passwords, with
  [MFA](../features/multi-factor-auth.md) for admins and
  [captcha](../features/captcha.md) on.
- **A team that already lives in Google Workspace, Okta or Azure AD:**
  [OAuth sign-in](../features/oauth-sign-in.md), and keep one password super
  admin for when the provider is down. The Okta and Azure AD templates need
  `oauth-pro`.
- **An enterprise customer with a SAML directory:** [SAML](../features/saml-sso.md)
  for sign-in and [SCIM](../features/scim-provisioning.md) for accounts.
- **Editors who sign in once a month:** [magic links](../features/magic-link-sign-in.md).

Programs do not sign in. A site or app uses an
[API key](../features/api-keys.md), and a script on the Admin API uses an
[admin token](../how-to/admin-tokens.md). See
[which credential do I need](../guides/user/roles-and-permissions.md#which-credential-do-i-need).
