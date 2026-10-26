---
title: User guide
description: "Find the shortest path to working in the LyEve admin console, and the page that holds each job an editor or administrator does there."
sidebar:
  order: 1
---

This guide is for the people who work in the LyEve admin console: editors who write and publish
content, and administrators who manage content types, users and roles. It needs no code. If you
run the server, see the [Operator guide](../operator/index.md). If you call the APIs from your
own code, see the [Developer guide](../developer/index.md).

## Signing in

![The console's sign-in form, with email, password, a forgotten password link and a sign-in link by email.](../../getting-started/images/sign-in.png)

Open the console at the address your operator gave you and sign in with your email and
password. If you have no account, ask an administrator to create one. Below the form you can
choose **Forgot password?** or **Email me a sign-in link**, and a **Sign in with** button
appears for each identity provider your instance is set up for. After several failed attempts
the form can ask you to complete a captcha.

A new install has no accounts yet. Its console opens a setup page that asks for the setup token
and creates the first administrator, as [Installation](../../getting-started/installation.md)
describes.

## Your first hour

1. [Core concepts](../../concepts/index.md): content types, entries, tenants and roles, the
   words every screen uses.
2. [Your first content type](../../getting-started/first-content-type.md): define a content type
   with a relation, preview the change and apply it. The **Schema builder** in the console does
   the same.
3. [Creating and editing content](content-editing.md): fill in an entry under **Content** and
   see what each field type expects.
4. [Drafts, publishing and revisions](drafts-and-publishing.md): publish, schedule, compare two
   revisions and restore an earlier one, when history is on for the collection.
5. [Media library](media-library.md): upload files, sort them into folders and attach them to
   entries.
6. [Users, roles and permissions](roles-and-permissions.md): who may do what, and how to give
   someone access.

## Common jobs

| I need to... | Read |
|---|---|
| Add or change a field on a content type | [Schema changes](../../features/schema-changes.md) |
| Start from a ready-made content model | [Schema presets](../../reference/schema-presets.md) |
| Write, save and publish an entry | [Creating and editing content](content-editing.md) |
| Publish at a set time, or undo a change | [Drafts, publishing and revisions](drafts-and-publishing.md) |
| Choose which collections keep history, and for how long | [Content history](../../features/content-history.md) |
| Send an entry through review before it goes live | [Editorial review](../../features/editorial-review.md) |
| Find entries once there are many | [Search and organization](search-and-organization.md) |
| Upload an image and use it in an entry | [Media library](media-library.md) |
| Load a spreadsheet of entries at once | [Bulk import](../../features/bulk-import.md) |
| Create an account or change someone's role | [Users, roles and permissions](roles-and-permissions.md) |
| Limit a role to some content types or fields | [Access rules](../../features/access-rules.md) |
| Give a website its own key | [API keys](../../features/api-keys.md) |
| Automate a task without code | [Flows](../../features/flows.md) |
| Export or erase one person's data | [Data protection](../../trust/data-protection.md) |

A screen appears in the sidebar only while the feature behind it runs for your tenant, and some
screens appear only for an `admin` or a `super_admin`. Working with entries in the console takes
the `admin` or `super_admin` role. The `editor` role writes through the Content API.

## Related

- [Features](../../features/index.md): every feature, and whether it is free.
- [Getting started](../../getting-started/index.md): what LyEve is and how to install it.
