---
title: Builder
description: "Design and publish your site's pages in the builder, with every site kept in your own engine, and see what the free tier holds."
sidebar:
  order: 3
---

> **Included free** on every install.

The builder is an app at [builder.lyeve.com](https://builder.lyeve.com) where you design your
site's pages, shared sections and theme, then publish them. It runs in your browser and keeps
nothing of its own. Every site and page is stored in your engine, in your tenant, and your site
reads the published pages through the Content API. The builder reaches your engine through a
builder connection that a person in your tenant approves, and it can do only what that person
may do.

:::caution[Beta]
The builder is in beta, and so is `builder-pro`, the capability that lifts the free tier's
ceiling.
:::

:::tip[Best for]
- **A site made of blocks.** Lay out pages from headings, text, images, card grids, galleries
  and forms, and keep a draft apart from what is live.
- **Starting from a sample.** Begin a site from the Blog, Marketplace, Portfolio or Business
  landing preset, each with its pages, two shared sections and a theme.
- **Pages that show your content.** List a content type's entries on a page, give each entry a
  page of its own, and take form posts into a content type or a flow.
- **One place for content and pages.** The pages live beside the rest of your content, under
  the same tenant, roles and backups.
:::

:::note[Use something else when]
- Your site is an app with its own templates and you only need its data. Read entries through
  the Content API, or serve them to anyone with
  [public content delivery](content-delivery.md).
:::

## Try it

You need an engine your browser can reach, and an account in it that is an editor, an admin or
a super admin.

**1. Turn the builder on.** The builder is off in every tenant until an administrator turns it
on. In LyEve Console, open **Settings > Builder connections**, choose the **Builder settings**
tab and turn on **Enabled**. The first time the builder is turned on in an engine, a super admin
has to do it, because that turn-on also writes the permission rules that let a builder
connection work on the builder's own content types. A turn-on by an admin before then saves the
switch and says that a super admin has to turn the builder on once.

**2. Connect the builder.** Open [builder.lyeve.com](https://builder.lyeve.com), enter your
engine's address under **Engine address** and choose **Connect**. The builder shows a code, such
as `ABCD-EF23`, and the fingerprint of the key it made in your browser. The code expires after
10 minutes.

**3. Approve the connection.** In LyEve Console, open **Settings > Builder connections**, choose
**Approve connection**, enter the code and choose **Find request**. Check that **Key** matches the
fingerprint the builder shows, then confirm with your authenticator code, or with your password
when your account has no authenticator app, and choose **Approve**.

**4. Make a site.** Back in the builder, create a site, empty or from a preset, and add a page.
Saving keeps a draft, which visitors never see. Publishing makes a page live once your site's
production build has reported the blocks it renders, as [Publish](../builder/publishing.md)
describes.

[Connect the builder](../builder/connect.md) covers every setting on the **Builder settings**
tab, the same steps with the API, and how to end a connection.

## Free and builder-pro

Every tenant keeps, at no charge:

| Ceiling | Free | Name in a refusal |
|---|---|---|
| Sites in the tenant | 2 | `builder.sites` |
| Pages in one site | 10 | `builder.site-pages` |
| Pages in the tenant, across its sites | 20 | `builder.pages` |

A site or a page counts from the moment it is created until it is deleted, drafts and archived
pages included. Shared sections and the theme do not count, and a preset's pages count like
pages you add by hand. A create past a ceiling is refused with `402` and names the ceiling it met
first:

```json
{
  "error": "cap_exceeded",
  "cap": "builder.sites",
  "limit": 2,
  "current": 2,
  "upgrade_url": "https://app.lyeve.com/pricing?skus=plugin-builder-pro&utm_campaign=builder.sites&utm_medium=product&utm_source=admin"
}
```

The `X-Upgrade-URL` header carries the same link as `upgrade_url`.

A license with the `builder-pro` feature lifts all three ceilings. It adds nothing else: the
editor, publishing, presets and every site you already have stay free. The engine reads the
license on every create, so a new key lifts the ceilings on the next one, with no restart. When
`builder-pro` lapses, every site and page you hold keeps its edit, publish, archive and delete,
and only the next create past a ceiling is refused. `builder-pro` is sold with the Pro, Growth
and Enterprise plans. See [pricing](https://lyeve.com/pricing/).

## Who may do what

| Role | What they can do |
|---|---|
| Viewer | Nothing on **Builder connections**, so a viewer cannot approve a connection |
| Editor | Approves connections, and lists and ends the ones they approved |
| Admin, super admin | Also turns the builder on, sets the **Builder settings** tab, lists and ends every connection of the tenant, and ends them all with **Revoke all** |

A connection lasts 30, 90 or 180 days, never longer than the tenant allows and never past the
expiry of the approver's own account. Turning **Enabled** off refuses the next call from every
connected browser.

## Related

- [The builder](../builder/index.md): what the builder makes and where it keeps it.
- [Connect the builder](../builder/connect.md): every setting, the API steps and ending a
  connection.
- [Sites and pages](../builder/sites-and-pages.md) and [Publish](../builder/publishing.md):
  working in the builder and making a change live.
- [AI assistant](ai-assistant.md): the builder's **Assistant** tab, which needs the `ai`
  feature.
