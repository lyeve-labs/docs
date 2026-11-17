---
title: "3. Model your content"
description: "Create the trip and trip_stop content types of the travel guide in the schema builder or over the Admin API, preview the database change, link them with a relation and turn on drafts."
sidebar:
  order: 3
covers:
  console:
    - "/admin/schema"
  api:
    - "POST /api/admin/schemas"
    - "POST /api/admin/schemas/{name}/preview-ddl"
    - "GET /api/admin/schemas/{name}/history"
    - "POST /api/admin/schemas/{name}/apply-pending"
    - "GET /api/admin/schemas/export"
    - "GET /api/admin/schemas/canvas-layout"
    - "PUT /api/admin/schemas/canvas-layout"
verified:
  engine: v0.62.0
  console: v0.28.0
---

A content type defines one kind of entry, its fields and the table that stores it. In this
lesson you create the two content types of the travel guide: `trip`, one journey, and
`trip_stop`, one place on it, linked to its trip. You need the super admin from
[lesson 1](run-lyeve.md) and a token in `TOKEN` for the API steps.

Saving a content type runs the database change at once, so these steps work on a fresh install.
On an instance that already holds a `trip` or a `trip_stop`, pick other names, because saving a
definition under a name that exists replaces it.

## The schema builder

> **Included free** on every install.

Open **Schema builder** in the sidebar. The list of content types sits on the left and the
editor on the right. The plus button at the start of the toolbar starts a new content type.

| Part | What it does |
|---|---|
| **Presets** | Creates a ready-made set of content types, such as a blog, in one step |
| **View** | Switches between **List** and **Canvas** |
| **Import and export** | **Export as YAML**, **Export as JSON**, and **Import** for a super admin |
| **Undo**, **Redo** | Step through your unsaved edits |
| **Preview DDL** | Shows the database statements a save would run, without running them |
| **Save** | Saves the content type and runs the change |
| **Display name**, **Machine name** | The label the console shows, and the name the APIs and the table use |
| **Table settings** | The system columns: `created_at`, `updated_at`, `soft_delete` |
| **Transports**, **Public delivery** | Where the content type is served, which lesson 11 uses |
| **Req**, **Idx**, **i18n**, **Private**, **No AI** | Per field: required, indexed, translated per locale, never served on public delivery, never read by the assistant |
| **Add field**, **Add relation** | Add a row to the definition |
| **Definition**, **API**, **History** | Tabs of a saved content type: its fields, a ready `curl` for it, and for a super admin the database changes each save ran |

A role that may read content types but not change them sees the same screen with the editing
controls turned off.

## Create the trip content type

> **Included free** on every install.

| Field | Type | Flags |
|---|---|---|
| `title` | `text` | **Req**, **i18n** |
| `slug` | `text` | **Idx** |
| `summary` | `rich_text` | **i18n** |
| `starts_on` | `date` | |
| `cover` | `media` | |

`title` and `summary` are marked as translated, which lesson 8 uses.

<div class="paths">
<div data-path="console">

1. Open **Schema builder** and choose the plus button at the start of the toolbar.
2. Enter `Trip` in **Display name** and `trip` in **Machine name**.
3. Choose **Add field**, type `title` as the field name, keep the type `text`, and tick **Req**
   and **i18n**.
4. Add `slug` the same way and tick **Idx**. Add `summary`, pick `rich_text` as its type and
   tick **i18n**. Add `starts_on` as a `date` and `cover` as a `media`.
5. Choose **Save**. The badge beside the name reads **Saved**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "trip",
        "display_name": "Trip",
        "fields": [
          { "name": "title", "field_type": "text", "required": true, "localized": true },
          { "name": "slug", "field_type": "text", "indexed": true },
          { "name": "summary", "field_type": "rich_text", "localized": true },
          { "name": "starts_on", "field_type": "date" },
          { "name": "cover", "field_type": "media" }
        ]
      }'
```

The answer is `200` with the stored definition, which now starts with the `id` field every
content type gets. A field without `field_type` answers `422`.

</div>
</div>

![A new Trip content type is built: the display and machine names are typed, five fields are added with their types and flags, and Save turns the badge to Saved.](./media/model-content-create-trip.webm)

[The data model](../concepts/data-model.md) lists every field type and flag.

## Preview a change before you save it

> **Included free** on every install.

Ask what a definition would do to the database before it does it. Here the change adds a
`country` field to `trip`.

<div class="paths">
<div data-path="console">

1. With **Trip** open, choose **Add field** and name it `country`.
2. Choose **Preview DDL**. Each statement shows the SQL a save would run, with the SQL that
   undoes it below.
3. Choose **Close**, then **Save**.

</div>
<div data-path="api">

Send the definition you would save, with the new field, to the preview route:

```bash
curl -X POST http://localhost:3001/api/admin/schemas/trip/preview-ddl \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "trip",
        "display_name": "Trip",
        "fields": [
          { "name": "title", "field_type": "text", "required": true, "localized": true },
          { "name": "slug", "field_type": "text", "indexed": true },
          { "name": "summary", "field_type": "rich_text", "localized": true },
          { "name": "starts_on", "field_type": "date" },
          { "name": "cover", "field_type": "media" },
          { "name": "country", "field_type": "text" }
        ]
      }'
```

The answer is `{"statements": [...]}`, each with a `description`, its `sql` and the `down_sql`
that undoes it. Nothing changes until you send the same body to `POST /api/admin/schemas`.

</div>
</div>

![Preview DDL opens a dialog with the add column statement for country and its down SQL, then Close returns to the editor.](./media/model-content-preview-ddl.webm)

A save that removes a field or changes a type that holds data asks first, in a dialog titled
**This save changes columns that hold data**. A removal is queued rather than run: the column
and its data stay until a super admin applies it from the **History** tab with **Apply**, or
with `POST /api/admin/schemas/{name}/apply-pending`. `GET /api/admin/schemas/{name}/history`
lists what each save ran and what is still queued. [Schema changes](../features/schema-changes.md)
walks a removal, a rename and a delete.

## Link each stop to its trip

> **Included free** on every install.

`trip_stop` holds the places on a trip. Its `trip` field is a relation: each stop belongs to
one trip.

| Field | Type | Flags |
|---|---|---|
| `name` | `text` | **Req** |
| `day` | `integer` | |
| `notes` | `rich_text` | |
| `trip` | `relation`, **Belongs to one**, pointing at **Trip** | |

<div class="paths">
<div data-path="console">

1. Choose the plus button again. Enter `Trip stop` in **Display name** and `trip_stop` in
   **Machine name**.
2. Add `name` as `text` with **Req**, `day` as `integer` and `notes` as `rich_text`.
3. Choose **Add relation** and name the field `trip`. In the panel under the row, keep
   **Belongs to one** under **Relation**, and pick **Trip** as the target. The panel says which
   table stores the key and what saving creates.
4. Choose **Save**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "trip_stop",
        "display_name": "Trip stop",
        "fields": [
          { "name": "name", "field_type": "text", "required": true },
          { "name": "day", "field_type": "integer" },
          { "name": "notes", "field_type": "rich_text" },
          { "name": "trip", "field_type": "relation", "relation_type": "belongs_to", "relation_to": "trip" }
        ]
      }'
```

A `belongs_to` field stores the other entry's id. Entries carry it as `trip_id` under `data`.

</div>
</div>

![A relation is added to Trip stop: Add relation makes a row named trip, Belongs to one stays selected, Trip is picked as the target, and the panel shows the key and what saving creates.](./media/model-content-relation.webm)

**How relations work**, folded under the fields, compares the four kinds: **Belongs to one**,
**Has one**, **Has many** and **Has and belongs to many**.
[The data model](../concepts/data-model.md) says what deleting an entry does to each.

## Turn on drafts and publishing

> **Included free** on every install.

Only the API turns drafts on. The schema builder in console v0.28.0 has no switch for them.
With drafts on, each trip carries a status, `draft`, `published` or `archived`, and the Content
API serves only published trips to readers. Lessons 4 and 6 depend on it.

Save `trip` again with `with_draft_publish`, keeping every field:

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "trip",
        "display_name": "Trip",
        "with_draft_publish": true,
        "fields": [
          { "name": "title", "field_type": "text", "required": true, "localized": true },
          { "name": "slug", "field_type": "text", "indexed": true },
          { "name": "summary", "field_type": "rich_text", "localized": true },
          { "name": "starts_on", "field_type": "date" },
          { "name": "cover", "field_type": "media" },
          { "name": "country", "field_type": "text" }
        ]
      }'
```

The answer is `200` with `"with_draft_publish": true`. A schema builder that had **Trip** open
during the call still holds the old definition and says so when you save. Choose **Reload**
there, then make your change again.

## Download the model

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Open **Import and export** in the toolbar and choose **Export as YAML** or **Export as JSON**.
The file holds every content type, ready to load on another instance with **Import**.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/schemas/export \
  -H "Authorization: Bearer $TOKEN" > travel-guide.yaml
```

</div>
</div>

[Moving content types between projects](../reference/content-type-portability.md) explains the
file and how an import treats names that exist. Lesson 9 moves the whole configuration.

## See the model on the canvas

> **Requires** a license with the `schema-pro` feature.

The canvas draws each content type as a box and each relation as a line, so `trip_stop` points
at `trip`. Pick a box to edit it in the same editor the list shows, and drag boxes to arrange
the diagram. The layout saves a moment after you stop, and every admin of the tenant sees it.

<div class="paths">
<div data-path="console">

1. In **Schema builder**, pick **Canvas** beside **List** under **View**.
2. Drag the **Trip stop** box below **Trip**. The arrangement is saved for the tenant.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/schemas/canvas-layout \
  -H "Authorization: Bearer $TOKEN"
```

The answer holds the saved positions. `PUT` on the same route saves them, and
[Schema changes](../features/schema-changes.md#work-on-the-schema-canvas) shows its body.

</div>
</div>

On a free install, **Canvas** shows what the canvas does and that your license does not include
it, and the list editor works as before. With the license, it draws the diagram.

![Choosing Canvas on a free install shows a notice about the schema canvas in place of the diagram.](./media/model-content-canvas-free.webm)

![Choosing Canvas with a license draws Trip and Trip stop joined by a line, and the Trip stop box is dragged below Trip.](./media/model-content-canvas-licensed.webm)

## Next

- [4. Write and publish](write-and-publish.md): write the first trips and stops.
- [Schema changes](../features/schema-changes.md): rename, remove and apply changes safely.
- [Your first content type](../getting-started/first-content-type.md): the same steps on a
  blog, with more on relations.
