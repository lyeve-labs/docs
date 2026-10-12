---
title: Your first content type
description: "Model a blog with Post and Author content types, preview and apply the change, then write, publish and read posts with their author."
sidebar:
  order: 5
---

This tutorial grows the `post` content type from the [Quickstart](quickstart.md) into a small
blog. You add an `author` content type, link each post to its author, turn on drafts and
publishing, and read a post back with its author in one call. Along the way you preview a
change before you apply it, the way you would on a live site.

You describe the shape of the content and the engine creates the storage for it in your
database. You never write SQL or a migration file, and the same definition works on
PostgreSQL, MySQL and SQL Server.

## Before you start

- `TOKEN` holds a token for an `admin` or `super_admin` account. Steps 2 and 3 of the
  [Quickstart](quickstart.md#2-create-the-first-administrator) get one. Changing content
  types takes one of those roles. Writing entries also works as an `editor`.
- The commands run from one directory, because step 2 saves a file there.
- You did step 4 of the Quickstart, or you start here: applying the definition in step 4
  below creates `post` when it does not exist yet.

The admin console's **Schema builder** does the same work in its list editor. Its **Preview DDL**
button shows the statements a save would run, and **Save** applies them. With Schema Pro the
same editor also opens from a visual canvas of every content type.

![The schema builder with the Post content type open: its display name, machine name and fields with their types.](images/schema-builder.png)

## 1. Add the Author content type

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "author",
        "display_name": "Author",
        "fields": [
          { "name": "name", "field_type": "text", "required": true },
          { "name": "bio", "field_type": "text" }
        ]
      }'
```

The answer is `200` with the stored definition. `display_name` is the label the admin console
shows. `required` refuses an author with no name.

## 2. Describe the new Post

Save this as `post-schema.json`. It keeps the Quickstart's two fields and adds three things:

```json
{
  "name": "post",
  "display_name": "Post",
  "with_draft_publish": true,
  "fields": [
    { "name": "title", "field_type": "text", "required": true },
    { "name": "body", "field_type": "rich_text" },
    { "name": "slug", "field_type": "text", "indexed": true },
    { "name": "author", "field_type": "relation", "relation_type": "belongs_to", "relation_to": "author" }
  ]
}
```

| What is new | What it does |
|---|---|
| `with_draft_publish` | Each post carries a status: `draft`, `published` or `archived` |
| `slug` with `indexed` | Finding a post by its slug stays fast as the blog grows |
| `author` | A `belongs_to` relation: each post points at one entry of `author` |

[The data model](../concepts/data-model.md) lists every field type, every flag such as
`unique` and `default`, the validation rules and the other kinds of relation.

## 3. Preview the change

Ask what the new definition would change before you apply it:

```bash
curl -X POST http://localhost:3001/api/admin/schemas/post/preview-ddl \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @post-schema.json
```

The answer is `{"statements": [...]}`, each with a `description`, the `sql` an apply would
run and the `down_sql` that undoes it. For this change you see a column for `slug` and its
index, a status column, and for the author a column, a step that clears links to authors
that do not exist, a foreign key and an index. Nothing changes
until you apply.

## 4. Apply it

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @post-schema.json
```

The answer is `200` with the definition as stored. The engine compares it with the stored
one and changes only what differs. Existing entries keep their values, and the Quickstart's
first post is now `published`. Applying the same file again changes nothing.

Removing a field later does not delete its data right away. The removal waits until a super
admin applies it, so a mistaken edit loses nothing. [The data model](../concepts/data-model.md)
covers renames and removals.

## 5. Write an author and a post

Create the author:

```bash
curl -X POST http://localhost:3002/api/v1/content/author \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"name": "Ada Lovelace", "bio": "Writes about engines."}}'
```

The answer is `201` with the new entry. Copy its `id`:

```bash
export AUTHOR_ID="<the id from the answer>"
```

Create a post that points at the author. A `belongs_to` field takes the other entry's `id`:

```bash
curl -X POST http://localhost:3002/api/v1/content/post \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "data": {
          "title": "Notes on the engine",
          "slug": "notes-on-the-engine",
          "body": "<p>A first look.</p>",
          "author": "'"$AUTHOR_ID"'"
        }
      }'
```

```bash
export POST_ID="<the id from the answer>"
```

The answer carries the link as `author_id` under `data`, and `_status` is `published`. A post
created through the Content API is always published, whatever the body says about its status.

## 6. Take it back to draft, then publish it

```bash
curl -X PUT http://localhost:3002/api/v1/content/post/$POST_ID/unpublish \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `204` with no body. The post is a draft now, so a plain list leaves it out:
`GET /api/v1/content/post` returns published entries only. To see the drafts, filter on the
status. The `-g` flag stops `curl` from reading the square brackets as a pattern:

```bash
curl -g "http://localhost:3002/api/v1/content/post?filters[_status]=draft" \
  -H "Authorization: Bearer $TOKEN"
```

Publish it again:

```bash
curl -X PUT http://localhost:3002/api/v1/content/post/$POST_ID/publish \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `204`, and the post is back in the list.

## 7. Read a post with its author

Find the post by its slug, and ask for the author to come with it:

```bash
curl -g "http://localhost:3002/api/v1/content/post?filters[slug]=notes-on-the-engine&populate=author" \
  -H "Authorization: Bearer $TOKEN"
```

```json
[
  {
    "id": "4769807e-2cbe-487b-be6f-1c87aabfbaa3",
    "schema_name": "post",
    "data": {
      "title": "Notes on the engine",
      "slug": "notes-on-the-engine",
      "body": "<p>A first look.</p>",
      "_status": "published",
      "author_id": "6c5b5d81-fa95-448d-ab1f-385bfad85209",
      "author": {
        "id": "6c5b5d81-fa95-448d-ab1f-385bfad85209",
        "name": "Ada Lovelace",
        "bio": "Writes about engines.",
        "created_at": "2026-10-02T05:06:55.656177Z",
        "updated_at": "2026-10-02T05:06:55.656177Z"
      }
    },
    "created_at": "2026-10-02T05:06:55.693943Z",
    "updated_at": "2026-10-02T05:07:03.696363Z"
  }
]
```

`GET /api/v1/content/post/$POST_ID` reads one post by its `id`, drafts included, and takes
`populate` too.

:::tip[A post with several authors]
Give `author` the `relation_type` `many_to_many` instead. The authors are then set with
`PUT /api/v1/content/post/{id}/relations/author` and a body of `{"ids": [...]}`, which
answers `204`, and read with `GET` on the same path. [The data model](../concepts/data-model.md)
describes each kind of relation and what deleting an entry does to it.
:::

## Next

- [Drafts and publishing](../guides/user/drafts-and-publishing.md): how editors work with
  drafts, revisions and scheduled publishing in the admin console.
- [Schema changes](../features/schema-changes.md): presets, renames and the history of every
  change.
- [Webhooks](../features/webhooks.md): tell another system when a post is published.
- [API endpoints](../reference/api-endpoints.md): paging, cursors and every other route that
  reads and writes entries.
