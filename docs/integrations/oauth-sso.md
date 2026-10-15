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
