---
title: Creating and editing content
description: "Create and edit entries in the admin console: pick a collection, fill in each kind of field, fix what a save refuses, and delete what you no longer need."
sidebar:
  order: 2
---

This guide is for people who write content in the LyEve admin console. It covers finding a
collection, creating an entry, filling in each kind of field and fixing what a save refuses.
You need no code. You do need the `admin` or `super_admin` role, because the content screens
are open to those roles only. See [Roles and permissions](roles-and-permissions.md).

A **collection** in the console is one content type, such as Article or Product, and an
**entry** is one item in it. Someone with the `admin` role defines the collection and its
fields in the **Schema builder**. What content types, fields and entries are is explained on
[the data model](../../concepts/data-model.md) page. If the **Content** screen says
**No schemas yet**, start with [Your first content type](../../getting-started/first-content-type.md)
or a [schema preset](../../reference/schema-presets.md).

## Find a collection

Open **Content** in the sidebar. The page lists every collection in your tenant, with its
**Rows**, **Fields** and **Last change**.

- Type in **Search by name or machine name** and press **Search** to narrow the list.
- Use the chips above the table, such as **With rows**, **Empty** or **Draft and publish**,
  to show only some collections.
- Open a collection to see its entries. Inside one, the **Collection** box at the top jumps to
  another.

![A collection's entries, with a column per field and each entry's status.](images/entry-list.png)

## Create an entry

1. Open **Content** and pick the collection.
2. Choose **New entry**.
3. Fill in the fields. A field marked `*` is required.
4. Choose **Create**.

You return to the collection's list, with the new entry at the top. In a collection that uses
draft and publish, the entry starts as a draft. See
[Drafts and publishing](drafts-and-publishing.md).

:::caution[A draft can already be live]
Today your site's Content API serves a console draft as published. The draft status shows in
the console only. Keep unfinished work out of collections your site reads until this is fixed.
:::

## How the console names an entry

The list and the entry page show an entry by its name, not its id. The console takes the name
from the entry's `title` field, then `slug`, then a field called `name`, `label`, `heading` or
`subject`. An entry with none of them shows as **Untitled entry** with the start of its id.

Every entry also has a slug, a short name that is unique across your tenant, even between
collections. When the collection has a `slug` field, the slug is what you type there. When it
does not, the console makes one from the name and adds six random characters, so two entries
called "Contact form" never collide.

## Fill in each kind of field

Each field shows an input that matches its type:

| Field type | What you see | What to enter |
|---|---|---|
| Text | A text box | A line or a paragraph |
| Rich text | A rich text editor | Formatted text. Unsafe markup, such as scripts and `javascript:` links, is removed when you save, so a pasted snippet can come back slightly changed. |
| Email | A box showing `user@example.com` | A valid address |
| URL | A box showing `https://` | A full link |
| Number | A number box | Digits, whole or decimal |
| Boolean | A checkbox | Checked for yes |
| Date | A date picker | A date without a time |
| Date and time | A date picker and a time | A date and a time |
| JSON | A JSON editor | Structured data as JSON |
| Media | A box for a link, and **Choose from library** | Pick a published file from the [media library](media-library.md), or paste the link of a file hosted elsewhere |
| Relation to one entry | A search box, such as **Search author** | Type to find the related entry and pick it |
| Relation to many entries | A list to pick from, such as **Select tag** | Pick every related entry |
| Relation stored on the other side | A read-only list | Nothing. It shows the entries that point here. Set the link on those entries. |
| Identifier | A read-only box, **Assigned on save** | Nothing. The server assigns it. |

When the related collection has no entries yet, a relation to one entry asks for the related
entry's id instead. Create the related entry first, such as the author before the post.

## When a save is refused

The console checks the form before it sends anything, and the server checks it again. If
something is wrong, nothing is saved and you see what to fix.

| Problem | What you see |
|---|---|
| A required field left empty | **This field is required** under the field |
| A value in the wrong format, such as an email address without `@` | A message naming the field |
| A slug another entry already uses | A message naming the collection that holds it, such as `slug is already taken by content type "page"` |
| A value outside the collection's rules | A message naming the field and the rule |

A collection can carry stricter rules: a pattern, a list of allowed values, a minimum or
maximum, or a rule that compares two number fields. A refused save never writes part of the
entry.

## Edit an entry

Open the entry from its collection, change the fields and choose **Save**. Each save is kept
as a revision, so you can look back at earlier versions and restore one. The entry page shows
when the entry was created and, if the collection keeps it, when it was last updated.
[Drafts and publishing](drafts-and-publishing.md) covers publishing and the revision history.

![An entry open in the editor: its status and Unpublish button, its history, the review card and its fields.](images/entry-editor.png)

When your team uses [editorial review](../../features/editorial-review.md), a **Review** card
above the form shows where the entry is in its review. When
[localization](../../features/localization.md) runs, a **Translations** card below the
form holds a tab per language. A **Release** card adds the entry to a
[release](../../features/content-releases.md) that publishes it with others, and a
**Comments** panel holds your team's [threads](../../features/editorial-comments.md) about
it. Adding to a release and starting a comment need a license with `content-pro`.

## Delete an entry

In the collection's list, choose the delete button at the end of the entry's row and confirm
with **Delete**. Its revisions and translations go with it.

## Where to go next

- [Drafts and publishing](drafts-and-publishing.md): publish, unpublish and restore earlier
  versions.
- [Media library](media-library.md): the files `media` fields point to.
- [Search and organization](search-and-organization.md): find entries once you have many.
- [Roles and permissions](roles-and-permissions.md): who may change collections and content.
