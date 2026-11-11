---
title: Show content entries
description: "List a content type's entries on a builder page, give each entry a page of its own, and draw a block's text, links and images from the entry."
sidebar:
  order: 6
---

LyEve Builder pages can show the entries of your content types: a list of entries on any page,
and a page for each entry. Your site reads them through the engine's public delivery routes,
with no key, so a page shows only published content that anyone may read.

## What a content type needs

A site's pages show entries only from content types that are on two lists. First, an admin or a
super admin opens the content type to the builder:

<div class="paths">
<div data-path="console">

1. Open **Settings**, choose **Builder connections**, then the **Builder settings** tab.
2. Under **Open to the builder**, tick the content types pages may show.
3. Choose **Save**.

</div>
<div data-path="api">

Name the content types in `show_schemas` on the builder's settings, at most 50. An empty list
clears it, and a field you leave out keeps its value:

```bash
curl -X PUT http://localhost:3001/api/admin/builder/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"show_schemas": ["product", "brand"]}'
```

</div>
</div>

Then the person editing the site chooses which of those content types the site uses. In the
builder, open **Site settings**. Under **Content**, which reads "Choose the content types this
site's pages can show.", tick the ones you want in **Content types**, then choose **Save**.

Up to eight content types show as checkboxes, and more as a list you can search. A site can
choose at most 50, and a choice past that says "Choose at most 50 content types." A content type
that is not open for public delivery says "Pages that show it cannot go live until an
administrator opens it for public delivery." A content type chosen earlier that an administrator
has since closed to the builder stays chosen, and the card names it.

When a list is empty, the builder says which step is missing:

| What is missing | What the builder says |
|---|---|
| No content type is open to the builder | No content type is open to the builder yet. Ask an administrator to open one in LyEve Console under Settings, then Builder connections. |
| The site chose none of the open content types | No content type is chosen for this site yet. Choose one in Site settings. |
| None of the chosen content types is open for public delivery | None of the content types chosen for this site is open for public delivery yet. Pages can show one only after an administrator opens it in LyEve Console. |

A published page also needs the content type open for public delivery, because that is the only
way your site reads it:

<div class="paths">
<div data-path="console">

Open the content type in **Schema builder**. Under **Public delivery**, tick **Serve on public
delivery**, then save the content type.

</div>
<div data-path="api">

A super admin sends the content type's definition with `"public_delivery": true` to
`PUT /api/admin/schemas/{name}`.

</div>
</div>

A field marked private is never served on public delivery, so the builder never offers it. The
builder offers lists, entry pages and fields only when your site's build reports that it draws
them, which every build of the builder's Astro starter does.

## List entries

The **Entry list** block lists pages by default. To list entries instead, open its **Source** in
**Properties**:

1. Under **List**, choose **Content type**, then the content type.
2. Under **Filters**, choose **Add filter** for each condition, pick a **Field** and enter the
   value it **Equals**.
3. Set **Entries to show**, from 1 to 48.

The list shows the newest entries first, and it has no other sort. **How many** applies only to
a list of pages.

A list takes up to four filters. Each matches one field to one value of up to 255 bytes, and
names a field only once. The value is written the way the field's type reads, such as `true` for
a `boolean` or `2026-10-01` for a `date`. A filter can name `id` or a field of type `text`,
`email`, `url`, `uid`, `integer`, `decimal`, `boolean`, `date` or `datetime`. A belongs-to
relation is filtered by its key column, such as `brand_id`, and the content type it points to
has to be open for public delivery too.

A site that draws pages on request shows "Could not load this list." in place of a list it cannot
read, and draws the rest of the page. A site built ahead of time stops its build instead, so the
last deploy stays live.

## Give each entry a page

An entry page draws one entry of a content type, at an address of its own. Open **Page settings**
on a page:

1. Under **Content type**, choose the content type.
2. Under **Path segment**, choose the field that names each entry in the address, or **Entry id**.
   Only a unique text field is offered.
3. End the page's **Path** in that segment in brackets, such as `/products/[slug]`, or in
   `[id]` for **Entry id**. A path that ends in anything else is refused with "End the path with
   [slug], such as /products/[slug]."

An address is a lower-case slug of up to 120 letters, digits and hyphens, or the entry's id.
Any other address, a draft, a deleted entry and an unknown address all answer your site's
not-found page.

On an entry page, **Search title** and **Search description** each offer **Written** or **From
the entry**. **Written** works as it does on [any page](sites-and-pages.md#search-title-and-description).
**From the entry** takes a field that can fill a text property, as the table under
[Draw a block from the entry](#draw-a-block-from-the-entry) lists. The title's hint then reads
"Shown in search results and the browser tab." A form on an entry page sends the visitor back
to that entry's address.

A site built ahead of time writes one page for each published entry, up to 1,000 for each entry
page, and its build log names the content type and how many entries were left out past that. A
site that draws pages on request can take up to two minutes to show an entry published or
withdrawn in the engine.

The builder's preview draws an entry page with the first published entry your site can show.
It never draws an unpublished entry. A note over the preview says what it drew, or why it drew
none:

| When | The note |
|---|---|
| An entry was drawn | Showing *title*. The title is the entry's title, name or heading field, the first that holds text |
| An entry was drawn, and it has no title, name or heading | Showing the first published entry of *content type*. |
| The content type has no published entry | No published *content type* entries yet. Publish one in LyEve Console to preview this page with it. |
| The content type is not open for public delivery | No entry to show, because *content type* is not open for public delivery. Ask an administrator to open it in LyEve Console. |
| The entry could not be read | Could not load an entry for the preview. |

The last note offers **Try again**. When no entry was drawn, every field from the entry is
empty, and a block that needs one shows "This *block* block shows entry fields, and the preview
has no entry to show." in its place, such as "This Headline block shows entry fields, and the
preview has no entry to show."

## Draw a block from the entry

On an entry page, a block property that can show a field of the entry offers **Draw from the
entry** in **Properties**. Choose the field, and the published page draws that field of each
entry in its place. **Remove binding** takes the field off the property.

A field inside a list item, such as one card of a card grid, offers **Draw from the entry** too.
The list says "Draw from the entry fills an item's field from the one entry this page shows. To
show a list of entries, use an Entry list block." Every item draws from that same entry, so use
an [Entry list](#list-entries) block to show many entries.

A block in a shared section cannot draw from an entry, because no one entry owns a section. A
page with no content type in **Page settings** cannot either.

| Property | Fields that fill it |
|---|---|
| Text, on one line or several | `text`, `uid`, `email`, `url`, `integer`, `number`, `decimal`, `date`, `datetime` |
| Rich text | `text`, `rich_text` |
| A number | `integer`, `number`, `decimal` |
| Yes or no | `boolean` |
| A link | `url` |
| An image | `media` |

No field fills a list itself, a choice, a form's destination or an entry list's source. A `json`
or `relation` field fills no property.

A `rich_text` field shows its headings, paragraphs, bulleted and numbered lists, quotes, bold,
italic, code and line breaks. A link shows as a link when its address is safe, and as plain text
when it is not. A table or any other part the page cannot show keeps only its text. Images,
embedded media and scripts are left out.

## What publish refuses

[Publish](publishing.md) checks what each page and section draws from your content, and the
first problem refuses the whole action with `422`:

| Code | Why |
|---|---|
| `page_invalid`, with the field code `field_not_listed` | A content type or field is not open to this site, a field is marked private, or a block draws from an entry outside an entry page. The answer names each place under `fields` |
| `page_invalid`, with the field code `schema_not_delivered` | A content type the page shows is not open for public delivery, or a filter or field names a relation into a content type that is not. Your site reads content only through public delivery, so the page would show nothing. Ask an administrator to turn on public delivery |
| `props_invalid`, with the rule `x-lyeve-prop` | A block property is bound to a field that cannot fill it, as the table above lists. The builder asks you to choose a field the property can show |
| `image_alt_missing` | An image is bound to a `media` field, and the content type has no `title`, `name` or `heading` text field |

A bound image takes its description from the entry's `title`, `name` or `heading` field, the
first that holds text. Without one, every image the page draws would have no description, so
give the content type one of those fields before you bind an image.

A save checks the same content types and fields, so the builder refuses a binding that names
one this site cannot show before you publish. Public delivery is checked only at publish, so an
administrator who turns it off blocks no one's save.

## Related

- [Take form posts](forms.md): put a form on a page, an entry page included.
- [Sites and pages](sites-and-pages.md): paths, drafts and versions.
- [Publish](publishing.md): the other checks a publish runs.
