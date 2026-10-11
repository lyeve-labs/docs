---
title: OAuth sign-in
description: Let your team sign in to the admin console with Google, Okta, Azure AD, Keycloak or any OpenID Connect provider instead of a password.
sidebar:
  order: 49
---

> **Included free** on every install, with one sign-in provider per install, added by hand or from the Google, GitHub and Apple templates. More providers, the Okta and Azure AD templates, role mapping and a backup provider need a license with the `oauth-pro` feature. See [pricing](https://lyeve.com/pricing).

OAuth sign-in puts a **Sign in with** button on the admin console's login page
for each identity provider you configure. Your team signs in with the account
they already use at work, and LyEve creates their console account the first
time they arrive.

:::tip[Best for]
- **A team on Google Workspace, Okta or Azure AD.** People sign in with the
  account their company already manages, and cannot sign in again once the
  company turns that account off. Okta and Azure AD need `oauth-pro`.
- **A self-hosted identity server.** Keycloak, Authentik, Zitadel or any other
  OpenID Connect provider, added by its issuer URL.
- **Fewer passwords to manage.** An account created through a provider starts
  with no password, so the LyEve login page refuses every password for it until
  the person sets one through [password reset](password-reset.md).
- **Staying up when a provider is down.** With `oauth-pro`, a backup provider
  takes over while the first one keeps failing.
:::

:::note[Use something else when]
- Your identity provider speaks SAML 2.0 only. Use
  [SAML single sign-on](saml-sso.md).
- You want accounts created and removed from your directory before anyone
  signs in. Add [SCIM provisioning](scim-provisioning.md).
- You want sign-in without any provider. [Magic link sign-in](magic-link-sign-in.md)
  is free.
:::

## How it works

| Step | What happens |
|---|---|
| Start | The browser opens `/api/admin/auth/oauth/{provider}`, and LyEve redirects it to the provider. |
| Sign in | The person signs in at the provider. |
| Callback | The provider sends the browser to `/api/admin/auth/oauth/{provider}/callback`. LyEve checks the answer and creates the account on a first visit. |
| Session | LyEve sets the session cookie and redirects to `/admin`. |

What you can rely on:

- The sign-in uses the authorization code flow with PKCE, a CSRF `state` and an
  OIDC `nonce`. The ID token's signature, issuer, audience and expiry are
  checked before anyone is signed in.
- A provider must report the email as verified. An unverified email is refused.
- A provider can never grant `admin` or `super_admin`. Those roles are dropped
  from whatever the provider sends.
- An existing user keeps the roles stored in LyEve. A provider's roles apply
  only when the sign-in creates the user.
- Client secrets are encrypted at rest and never returned by any route.
- A session from OAuth sign-in lasts `JWT_EXPIRY_SECS` and comes with no
  refresh token, so the person signs in again when it ends.

## Turn it on

1. Check what your tier allows in [free and oauth-pro](#free-and-oauth-pro).
2. Set `LYEVE_BASE_URL` to the public address of the instance, such as
   `https://lyeve.example.com`. Every callback address is built from it, never
   from the request's `Host` header.
3. Set `SECURE_COOKIE=true` when the instance is served over HTTPS.
4. Restart the instance, then add a provider. A super admin opens **Access >
   OAuth providers** in the admin console and chooses **New provider**, or uses
   the API below.

## Try it

This adds Google from its template. You need a super admin token in `TOKEN`.
The [quickstart](../getting-started/quickstart.md) shows how to get one.

1. List the templates:

   ```bash
   curl http://localhost:3001/api/admin/oauth-templates \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer lists five templates (`google`, `github`, `okta`, `azuread` and
   `apple`), each with the `fields` it needs.

2. Read the Google template and the callback address to register:

   ```bash
   curl http://localhost:3001/api/admin/oauth-templates/google \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer holds the `template` and a `redirect_uri` such as
   `https://lyeve.example.com/api/admin/auth/oauth/google/callback`. Register
   that address in the Google Cloud console. Without `LYEVE_BASE_URL` the
   answer is `500` with `OAuth is not configured: set base_url in configuration`.

3. Apply the template with your client's values:

   ```bash
   curl -X POST http://localhost:3001/api/admin/oauth-templates/apply \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"key": "google", "overrides": {"client_id": "1234.apps.googleusercontent.com", "client_secret": "GOCSPX-..."}}'
   ```

   ```json
   {
     "provider": {
       "id": "1f04498a-c8d8-48f0-8165-8001a04a20b3",
       "name": "google",
       "client_id": "1234.apps.googleusercontent.com",
       "issuer_url": "https://accounts.google.com",
       "scopes": ["openid", "email", "profile"],
       "roles_claim": "roles",
       "default_roles": ["editor"],
       "enabled": true,
       "created_at": "2026-10-01T09:30:00Z",
       "updated_at": "2026-10-01T09:30:00Z"
     },
     "redirect_uri": "https://lyeve.example.com/api/admin/auth/oauth/google/callback",
     "template": "google"
   }
   ```

   The answer is `201`. A provider made from a template is enabled unless
   `overrides` holds `"enabled": false`.

4. Check what the login page will offer:

   ```bash
   curl http://localhost:3001/api/admin/auth/oauth-providers
   ```

   ```json
   [{ "name": "google" }]
   ```

5. Open the admin console's login page. Under **or continue with** it shows a
   sign-in button for Google. Providers are listed under **Access > OAuth
   providers**, which only a super admin sees.

## Free and oauth-pro

| | Free | With `oauth-pro` |
|---|---|---|
| Providers | 1 per install, enabled or not | Unlimited |
| Templates | `google`, `github` and `apple`, or any OpenID Connect issuer by hand | Also `okta` and `azuread` |
| Roles | The `roles` claim and the `editor` default role | Any `roles_claim` and any `default_roles` |
| Failover | Health checks and the circuit reset | Also a `backup_provider_id` |

Providers belong to the instance, because the callback address is the
instance's, so every tenant shares the one free provider. Updating, disabling
and deleting a provider, the health view and every account link route are free.
The license is read on every request. A provider set up with `oauth-pro` keeps
signing people in with what it stored after a license lapses, and only a new
provider or a change to a paid option is refused. A second provider on a free
install answers:

```json
{"error": "cap_exceeded", "cap": "oauth.providers", "limit": 1, "current": 1, "upgrade_url": ""}
```

A paid template or option answers `402` with
`{"error": "payment_required", "plugin": "oauth", "feature": "feature:oauth-pro", "upgrade_url": ""}`.

## Choose a template

| Template | Fields in `overrides` |
|---|---|
| `google` | `client_id`, `client_secret` |
| `github` | `client_id`, optional `client_secret` |
| `okta` | `domain`, `client_id`, `client_secret`, optional `authorization_server`. Needs `oauth-pro`. |
| `azuread` | `tenant_id` (a directory id, `common` or `consumers`), `client_id`, `client_secret`. Needs `oauth-pro`. |
| `apple` | `client_id` (the Service ID), `client_secret`, `team_id`, `key_id` |

Every template also takes `name`, to save the provider under another name, and
`enabled`. With `name` set, register the `redirect_uri` from the apply answer,
because the callback path uses the name.

:::caution
- The `google` template's `hosted_domain` and the `github` template's `org`
  are accepted but discarded before the provider is saved. They do not limit
  who can sign in.
- Apple expects a client secret that is a token signed with your private key.
  The `apple` template asks for the private key and says it builds that token,
  but it does not: it sends the value of `client_secret` as it is, and
  `team_id` and `key_id` are discarded. Sign-in fails at the token exchange
  with `502 token exchange failed`.
:::

## Add any OpenID Connect provider

```bash
curl -X POST http://localhost:3001/api/admin/oauth-providers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "keycloak",
    "client_id": "lyeve-admin",
    "client_secret": "s3cr3t",
    "issuer_url": "https://sso.example.com/realms/staff",
    "roles_claim": "roles",
    "default_roles": ["editor"],
    "enabled": true
  }'
```

The answer is `201` with the saved provider. Register
`{LYEVE_BASE_URL}/api/admin/auth/oauth/{name}/callback` as the redirect
address with the provider.

<details>
<summary>Provider fields</summary>

| Field | Meaning | Default |
|---|---|---|
| `name` | The provider's name in the sign-in URL. Lowercased, spaces become `_` | required |
| `client_id`, `client_secret` | The client registered with the provider | `client_id` required |
| `issuer_url` | The provider's OIDC issuer. An address that is private, internal or does not resolve is refused | required |
| `scopes` | Scopes to request | `["openid", "email", "profile"]` |
| `roles_claim` | The ID token claim that carries roles. Another claim needs `oauth-pro` | `roles` |
| `default_roles` | Roles a new user gets when the provider sends none. Other roles need `oauth-pro` | `["editor"]` |
| `enabled` | Whether the login page offers it | `false` |
| `backup_provider_id` | Another provider to send sign-ins to while this one is failing. Needs `oauth-pro` | none |

A `PUT` changes only the fields you send, so the stored secret stays unless
you send a new one.

</details>

## When an email already has an account

When a provider identity signs in for the first time and a user with the same
email already exists, the sign-in is refused with `409`. LyEve never links an
existing account on its own, because someone holding that email at another
provider could otherwise take the account over. Sign in through a provider
before creating a password account for the same person, or remove the
password account first.

A link ties one LyEve user to one identity at one provider. Links belong to a
tenant, and a user can hold several. A super admin reads them under
`/api/admin/account-links` and can unlink one, make one primary, or move an
identity to another user.

<details>
<summary>Manage account links</summary>

Unlinking takes a CSRF token. Read the link first, which sets a `csrf_token`
cookie, then send the same value back as the cookie and the `X-CSRF-Token`
header:

```bash
curl -c jar -H "Authorization: Bearer $TOKEN" \
  http://localhost:3001/api/admin/account-links/$LINK_ID
curl -b jar -X DELETE -H "Authorization: Bearer $TOKEN" \
  -H "X-CSRF-Token: $(awk '/csrf_token/ {print $7}' jar)" \
  http://localhost:3001/api/admin/account-links/$LINK_ID
```

```json
{ "message": "unlinked" }
```

A user's last link cannot be removed: that answers `409`.

`PATCH /api/admin/account-links/{id}/primary` with `{"user_id": "<user id>"}`
makes a link primary and answers `{"message": "primary set"}`. A link made at
sign-in is not primary until you do this.

`POST /api/admin/account-links/resolve` moves an identity to another user:

```json
{
  "provider_name": "google",
  "provider_sub": "109876543210",
  "target_user_id": "5f0c7b1e-1d2a-4c3b-9e8f-0a1b2c3d4e5f",
  "source_user_id": "8a9b0c1d-2e3f-4a5b-8c7d-6e5f4a3b2c1d"
}
```

The answer is `{"message": "conflict resolved"}`.

:::danger
Resolve moves every link the source user holds to the target user, then
deletes the source user.
:::

</details>

## Keep sign-in up when a provider fails

Each enabled provider's discovery address is checked every 30 seconds.
`GET /api/admin/oauth-providers/health` returns a `providers` list, one entry per provider, with
`healthy`, `latency_ms`, `last_checked_at`, `last_error` and a `circuit` whose
`state` is `closed`, `open` or `half_open`.

After 5 failures within 60 seconds a provider's state turns `open`. While it
is open, a sign-in through it goes to its `backup_provider_id` (set with
`oauth-pro`) when that
provider is enabled, and to the provider itself when there is no backup. After
30 seconds the next sign-in tries the provider again, and a success closes the
circuit. `POST /api/admin/oauth-providers/{id}/circuit/reset` sets it back to
`closed` at once.

## Settings

| Variable | What it does | Default |
|---|---|---|
| `LYEVE_BASE_URL` | Public address the callback is built from. Required. | unset |
| `SECURE_COOKIE` | Marks the sign-in and session cookies `Secure`. Set `true` behind HTTPS. | `false` |
| `JWT_EXPIRY_SECS` | Lifetime of the session a sign-in creates. | `900` |
| `OAUTH_HEALTH_CHECK_TIMEOUT` | Whole seconds the health check waits for a provider. | `5` |

If you set `LYEVE_PLUGINS`, include `oauth` in it.

## Routes

Every route that needs a role takes a signed-in session. An admin token is
refused on them with `403`.

<details>
<summary>Sign-in, providers, templates and links</summary>

| Method | Path | Role | Purpose |
|---|---|---|---|
| `GET` | `/api/admin/auth/oauth-providers` | public | Names of enabled providers, for login buttons |
| `GET` | `/api/admin/auth/oauth/{provider}` | public | Start sign-in. Redirects the browser to the provider |
| `GET` | `/api/admin/auth/oauth/{provider}/callback` | public | Where the provider sends the browser back |
| `GET` | `/api/admin/oauth-providers` | `admin` | List providers, paginated |
| `POST` | `/api/admin/oauth-providers` | `super_admin` | Add a provider. `201` |
| `PUT` | `/api/admin/oauth-providers/{id}` | `super_admin` | Change a provider |
| `DELETE` | `/api/admin/oauth-providers/{id}` | `super_admin` | Remove a provider. `200` with `{"message": "deleted"}` |
| `GET` | `/api/admin/oauth-providers/health` | `admin` | Whether each provider is reachable |
| `POST` | `/api/admin/oauth-providers/{id}/circuit/reset` | `super_admin` | Mark a failing provider usable again |
| `GET` | `/api/admin/oauth-templates` | `admin` | List the templates |
| `GET` | `/api/admin/oauth-templates/{key}` | `admin` | Read one template and its callback address |
| `POST` | `/api/admin/oauth-templates/apply` | `super_admin` | Add a provider from a template. `201` |
| `GET` | `/api/admin/account-links?user_id={id}` | `admin` | The provider identities linked to a user |
| `GET` | `/api/admin/account-links/{id}` | `admin` | Read one link |
| `DELETE` | `/api/admin/account-links/{id}` | `super_admin` | Unlink an identity |
| `PATCH` | `/api/admin/account-links/{id}/primary` | `super_admin` | Make a link the user's primary one |
| `POST` | `/api/admin/account-links/resolve` | `super_admin` | Move an identity's links to another user |

</details>

## Errors

| Status | Message | Cause |
|---|---|---|
| `500` | `OAuth is not configured: set base_url in configuration` | `LYEVE_BASE_URL` is not set. |
| `403` | `email address is not verified by the identity provider` | The provider sent an unverified email. |
| `409` | `an account with this email already exists; please log in with your existing credentials and link this provider from your account settings` | See [when an email already has an account](#when-an-email-already-has-an-account). |
| `502` | `failed to reach identity provider` or `token exchange failed` | The provider is down or the client credentials are wrong. |

<details>
<summary>Every other error</summary>

| Status | Message | Cause |
|---|---|---|
| `404` | `provider not found` | No enabled provider has that name. |
| `400` | `state mismatch: possible CSRF` | The browser came back with another sign-in's state. |
| `400` | `missing oauth state` | The sign-in cookies expired or were blocked. They last ten minutes. |
| `400` | `oauth error: <code>` | The provider refused the sign-in and said why. |
| `401` | `authentication failed` | The account is disabled. |
| `403` | `identity token validation failed` | The ID token did not verify. |
| `422` | `name, client_id, and issuer_url are required` | A provider was created without them. |
| `422` | `template application failed` | A template was applied without one of its required fields. |
| `422` | `base_url is required for redirect URI computation: set base_url in configuration or provide it in the request body` | A template was applied with no base URL. |
| `400` | `issuer_url is not allowed (blocked destination)` | The issuer is private, internal or does not resolve. |
| `409` | `cannot unlink last account: user must have at least one linked provider` | Unlinking a user's only link. |
| `402` | `cap_exceeded` | A second provider without `oauth-pro`. |
| `402` | `payment_required` | The Okta or Azure AD template, role mapping or a backup provider without `oauth-pro`. |

</details>

## Related

- [Sign-in options](../integrations/oauth-sso.md): every way to sign in, side
  by side.
- [SAML single sign-on](saml-sso.md): for SAML 2.0 identity providers.
- [Multi-factor authentication](multi-factor-auth.md): a second step for
  password sign-in.
- [Validate tokens with JWKS](../integrations/validate-tokens-jwks.md): check
  the session token in your own services.
