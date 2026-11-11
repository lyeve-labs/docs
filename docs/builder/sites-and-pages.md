---
title: Sites and pages
description: "Make sites in LyEve Builder, start one from a preset, give each page a path, share sections between pages and keep a page's earlier versions."
sidebar:
  order: 3
---

LyEve Builder keeps each site in your engine as a set of entries: the site, its pages, its
shared sections and its theme. This page says what each one holds and the rules a save follows.
The builder signs every call with its connection, so what you may do here is what your role may
do. A role that may not change a site's pages is refused with "Your role cannot edit this site."

## Sites

A site starts empty or from a preset. A preset is a sample site with its pages, shared sections
and a theme, created together, with every page a draft. On the free tier a preset needs room
for one more site and for all of its pages, or nothing is created. See
[what is free](index.md#what-is-free).

A site you no longer use can be archived and restored. A site is deleted only once it holds no
pages and no sections, so archive one that still has them.

## Pages

Each page has a title and a path. The path is unique in its site and follows these rules:

- It is at most 255 bytes long.
- It holds no percent sign, question mark, hash, backslash or invisible character.
- It holds no `..`, no part that is only `.`, and no two slashes in a row.
- It is not a path the site serves itself.

A page keeps a draft apart from its live version. Saving changes the draft only, so what
visitors see changes when you [publish](publishing.md). You can discard a draft to go back to
what is live. A published page cannot be deleted: unpublish or archive it first. An archived
page can be restored.

A page holds at most 1,000 KiB, and each rich text value in it at most 256 KiB. A save the page
format refuses names each field it refused, at most 50 of them.

### Search title and description

Every page has a **Search title** and a **Search description** in **Page settings**. A search
title holds up to 255 characters, and a description up to 500. Each is kept on one line, so a
line break you type becomes a space. A longer value is refused with "Enter a search title of 255
characters or fewer." or "Enter a search description of 500 characters or fewer."

The search title's hint reads "Leave it empty to show the page title in search results and the
browser tab." The description's hint reads "Shown under the title in search results." On an
[entry page](content.md#give-each-entry-a-page), each can come from a field of the entry instead.

### Versions

While [content history](../features/content-history.md) is on for the builder's drafts, the
`lyeve_builder_drafts` content type, every save of a page's draft is kept as a version. You can
save a draft as a named version, list a page's versions, see what changed in one and restore
it.

### Translations

A site has a default locale. A translation is a copy of a page or section in the default
locale, made once for each other locale, and it keeps a link to the page it came from.

## Sections

A section is a part you place on many pages, such as a header or a footer. Changing it changes
every page that places it once you publish. A section that is placed on pages cannot be
deleted, so remove it from those pages first. A section can be archived and restored.

## The theme

Each site has one theme: the colors and fonts it draws with. Your site's own build can report
default colors, which the builder shows until the theme sets its own.

## Free ceiling

Without the `builder-pro` feature a tenant keeps 2 sites, 10 pages in each site and 20
pages in all. A page moved from a full site to another never adds room, because the tenant's
total counts too. Archived pages and drafts count until they are deleted.
[What is free](index.md#what-is-free) lists the refusal each ceiling answers.

## Related

- [Publish](publishing.md): make a draft live and rebuild the site.
- [Connect the builder](connect.md): who may approve a connection, and for how long.
