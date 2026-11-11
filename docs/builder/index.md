---
title: LyEve Builder
description: "Learn what LyEve Builder does, where your sites and pages are kept, and how many sites and pages are free."
sidebar:
  order: 1
---

LyEve Builder: design and publish your site's pages, while the content stays in your own
engine.

The builder runs in your browser and keeps nothing of its own. Each site, page, shared section
and theme you make in it is stored in your engine, in your tenant, as entries of content types
the builder owns. Your site reads the published pages through the engine's Content API, the
same way it reads the rest of your content.

## How the builder reaches your engine

The builder holds no password and no API key. When you connect it, it makes a key in your
browser and asks your engine to accept it. A person in your tenant approves that key, and from
then on every call the builder makes is signed with it. The engine checks each signature, and
the builder may do only what the person who approved it may do.

[Connect the builder](connect.md) walks through turning the builder on, approving a builder
connection and ending one.

## What you make

- **Sites**: one per website. A site starts empty or from a preset, a sample site with its
  pages, shared sections and a theme.
- **Pages**: each at its own path in its site. A page keeps a draft apart from what is live,
  so saving never changes what visitors see.
- **Sections**: parts you place on many pages, such as a header or a footer.
- **The theme**: the colors and fonts the site draws with.

[Sites and pages](sites-and-pages.md) covers each of them, and
[Publish](publishing.md) covers making a change live and rebuilding a static site.

A page can also draw from your engine and post to it.
[Show content entries](content.md) lists a content type's entries and gives each entry a page,
and [Take form posts](forms.md) puts a form on a page that saves each post in a content type or
starts a flow.

## What is free

Every tenant may keep, at no charge:

| Ceiling | Free |
|---|---|
| Sites in the tenant | 2 |
| Pages in one site | 10 |
| Pages in the tenant, across its sites | 20 |

A site or page counts from the moment it is created until it is deleted, drafts and archived
pages included. Sections and the theme do not count. A create past a ceiling is refused with
`402` and the code `cap_exceeded`, which names the ceiling it met and how many are in use:

| Ceiling name | Means |
|---|---|
| `builder.sites` | The tenant holds 2 sites |
| `builder.site-pages` | The site holds 10 pages |
| `builder.pages` | The tenant holds 20 pages |

A license with the `builder-pro` feature lifts all three, and the refusal's `upgrade_url`
links to the page where you can buy it.

## Related

- [LyEve Console](../console/index.md): where the people who approve the builder sign in.
- [Webhooks](../features/webhooks.md): what calls your host's deploy hook after a publish.
