---
title: Glossary
description: "Look up what a LyEve term means where you meet it in the admin console, the APIs and these docs."
sidebar:
  order: 4
---

**Access rule**
A rule that says what one role may do with a content type, a flow or a review, and which
fields a read hides from it. See [Access rules](../features/access-rules.md).

**Admin API**
The HTTP API on port `3001`, under `/api/admin/`, that manages the instance: sign-in, content
types, users, settings. See [the two APIs](../concepts/index.md#the-two-apis).

**Admin console**
The web interface for editors and administrators. It signs you in and calls the Admin API for
you, so nothing it does is out of reach of a script. See the [User guide](../guides/user/index.md).

**Admin token**
A credential for scripts that call the Admin API, of the form `lyat_...`. It acts in one
tenant, opens only the routes its grants name, and expires within 90 days. See
[Admin tokens](../how-to/admin-tokens.md).

**API key**
A credential for programs that call the Content API, of the form `ly_...`, sent in the
`X-API-Key` header and limited to the roles and scopes you grant it. See
[API keys](../features/api-keys.md).

**Capability**
A paid part of a free feature, licensed on its own. Flows are free, and `flow-pro` lifts
their limits. See [Licensing and tiers](../concepts/licensing-and-tiers.md).

**Content API**
The HTTP API on port `3002`, under `/api/v1/`, that your sites, apps and services use to read
and write entries. See [the two APIs](../concepts/index.md#the-two-apis).

**Content type (schema)**
The shape of one kind of content: a name, its fields and options such as draft and publish.
The API calls it a schema, so both words mean the same thing here. See
[The data model](../concepts/data-model.md).

**Database dialect**
Which database an install runs on: PostgreSQL, MySQL or SQL Server. The scheme of
`DATABASE_URL` decides it. See [Installation](../getting-started/installation.md#choose-a-database).

**Draft and publish**
An option on a content type. Its entries then carry a status of `draft`, `published` or
`archived`, and a list on the Content API holds published ones unless the request filters on
the status. See
[The data model](../concepts/data-model.md).

**Entitlements**
What your license grants, as `GET /api/admin/entitlements` reports it: plan, state, the
features that run, limits, expiry and grace period. The license itself is never shown. See
[Licensing and tiers](../concepts/licensing-and-tiers.md).

**Entry**
One item of a content type, such as one post or one product, with its values under `data`.
See [The data model](../concepts/data-model.md).

**Feature**
A part of the product that can run or not, such as webhooks, search or multi-factor
authentication. Some are included free on every install and the rest need a license that
names them. See [Features](../features/index.md).

**Field**
One value in a content type, with a `field_type` such as `text`, `number` or `relation` and
flags such as `required` or `unique`. See [The data model](../concepts/data-model.md).

**Flow**
An automation or custom endpoint drawn on a canvas: a trigger, then the steps it runs. See
[Flows](../features/flows.md).

**Free tier**
An install with no license. Every feature marked "Included free" runs on it, with no time
limit. See [Licensing and tiers](../concepts/licensing-and-tiers.md).

**JWKS**
JSON Web Key Set, served at `GET /.well-known/jwks.json` on the Content API with no sign-in.
It publishes the engine's public signing key, so another service can check a LyEve session
token without a shared secret.

**License key**
The value of `LYEVE_LICENSE_KEY`. Either a signed token, checked on the instance with no
network call, or a key starting `lyeve_live_` that the engine exchanges for a token. See
[Licensing and tiers](../concepts/licensing-and-tiers.md).

**Preset**
A ready-made set of content types created in one call. Once created, the content types are
yours to change. See [Schema presets](schema-presets.md).

**Relation**
A field that links an entry to entries of another content type, such as a post to its author.
See [The data model](../concepts/data-model.md).

**Revision**
A stored earlier version of an entry. Every save in the admin console keeps one, and you can
compare and restore them. See [The data model](../concepts/data-model.md).

**Role**
A name on an account or a key that decides what it may do. Three are built in: `super_admin`,
`admin` and `editor`. See [Roles and permissions](../guides/user/roles-and-permissions.md).

**Setup token**
The one-time secret that creates the first administrator, printed to the engine's log or set
in `LYEVE_SETUP_TOKEN`. See the [Quickstart](../getting-started/quickstart.md#2-create-the-first-administrator).

**Super admin**
The highest role. It can act in any tenant with the `X-Tenant-ID` header, and the routes that
affect the whole instance require it. See
[Roles and permissions](../guides/user/roles-and-permissions.md).

**Tenant**
An isolated customer, brand or site within one install, named by a slug of lower-case
letters, digits and `_` that starts with a letter, at most 63 characters. A session and an
API key each act in one tenant and see only its data. A user can belong to several. See
[Tenants](../features/tenants.md).

## See also

- [Core concepts](../concepts/index.md): how these terms fit together.
- [API endpoints](api-endpoints.md): where these terms appear in the HTTP APIs.
