---
title: Publish
description: "Publish pages, sections and the theme from LyEve Builder as one action, see why a publish is refused, and rebuild a static site through its host's deploy hook."
sidebar:
  order: 4
---

A publish in LyEve Builder copies drafts into their live entries, so the Content API serves
them. One action can publish pages, shared sections and the theme together, at most 50 of them.
An unpublish takes pages back to draft, and the Content API stops serving them.

## What a publish checks first

The whole action is checked before anything is written. The first problem refuses it, and
nothing is published:

| Code | Why |
|---|---|
| `permission_denied` (`403`) | Your role may not change one of the kinds the action names |
| `entry_not_found` (`404`) | An item is not a page, section or theme of this site |
| `page_invalid` (`422`) | A draft breaks the page format. The answer names each field |
| `registry_missing` (`422`) | Your site's production build has not reported its blocks yet |
| `format_not_rendered` (`422`) | Your production site does not read this page's format |
| `block_version_unknown` (`422`) | A block, or a block version, is newer than your production site renders |
| `image_alt_missing` (`422`) | An image has no description and is not marked decorative |
| `section_unpublished` (`422`) | A placed section is deleted, belongs to another site, or is not published and not in this action |
| `store_unavailable` (`503`) | A read failed before anything was written. Try again after the `Retry-After` time |

The block checks compare the page with what your site's production build reported it renders.
A site rolled back to an older build refuses pages that hold newer blocks until it is deployed
again. A lower block version publishes, because the site upgrades it when it reads the page.

Each item then answers `applied`, `unchanged` when its live entry already matches, or `failed`
with a reason. A failure stops the items after it, and the answer then says
`complete: false`. The builder
names each action with an id, so an action sent again after a timeout returns the first answer
and starts no second build.

## Rebuild a static site

A static site shows a change only after its host builds it again, which the host does when its
deploy hook is called. Each publish or unpublish that changed something creates one entry of
the `lyeve_builder_publishes` content type, and a [webhook](../features/webhooks.md) on that
content type calls the deploy hook. A draft save and a settings change create none, so they
start no build.

LyEve Console's **New webhook** form lists only content types you made, so a super admin
creates this one with the API:

```bash
curl -X POST http://localhost:3001/api/admin/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Rebuild the marketing site",
        "url": "<the deploy hook URL your host gives you>",
        "events": ["after_create"],
        "schemas": ["lyeve_builder_publishes"],
        "field_filters": {"site_id": "<the site id>"}
      }'
```

| Field | Value | Why |
|---|---|---|
| `events` | `["after_create"]` | One call for each publish or unpublish |
| `schemas` | `["lyeve_builder_publishes"]` | No other content type calls the hook |
| `field_filters` | `{"site_id": "<the site id>"}` | Only this site's actions call it, in a tenant with more than one site |

A field filter needs a license with the `webhook-pro` feature. Without one, leave
`field_filters` out, and every publish of any site in the tenant calls every such webhook. A
tenant with one site loses nothing.

With no webhook, a publish still succeeds. A site that renders each request shows the change at
once, and a static site keeps its last build. A failed call is retried, and the console's
**Webhooks** screen shows each delivery.

A page published another way, such as from the console, by a schedule or by a review approval,
starts no build. After one of those, publish once from the builder, or rebuild the site from
its host.

## See past publishes

The builder lists a site's publishes and unpublishes, newest first, 50 to a page, each with its
items, whether it completed and who made it.

## Related

- [Sites and pages](sites-and-pages.md): drafts, versions and the page format.
- [Webhooks](../features/webhooks.md): retries, delivery history and signing.
