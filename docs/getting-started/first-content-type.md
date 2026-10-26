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

- You did step 4 of the Quickstart, or you start here: applying the definition in step 4
  below creates `post` when it does not exist yet.
- Changing content types takes the `admin` or `super_admin` role. Writing entries also works
  as an `editor`.
- For the console steps, you are signed in to the admin console. Its **Schema builder** edits
  each content type in a list. With Schema Pro the same editor also opens from a visual canvas
  of every content type.
- For the API steps, `TOKEN` holds a token for an `admin` or `super_admin` account. Steps 2
  and 3 of the [Quickstart](quickstart.md#2-create-the-first-administrator) get one. The
  commands run from one directory, because step 2 saves a file there.

## 1. Add the Author content type

<div class="paths">
<div data-path="console">

1. Open **Schema builder** and choose **New schema**.
2. Enter `Author` in **Display name** and `author` in **Machine name**.
3. Choose **Add field**, name it `name`, keep the type `text` and tick **Req**.
4. Choose **Add field**, name it `bio` and keep the type `text`.
5. Choose **Save**.

</div>
<div data-path="api">

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

The answer is `200` with the stored definition.

</div>
</div>

The display name is the label the admin console shows. A required `name` refuses an author
with no name.

## 2. Describe the new Post

The new Post keeps the Quickstart's two fields and adds two:

| What is new | What it does |
|---|---|
| `slug`, indexed | Finding a post by its slug stays fast as the blog grows |
| `author` | A `belongs_to` relation: each post points at one entry of `author` |

<div class="paths">
<div data-path="console">

1. In **Schema builder**, choose **Post** in the list of content types.
2. Choose **Add field**, name it `slug`, keep the type `text` and tick **Idx**.
3. Choose **Add relation** and name the field `author`. Under **Relation**, keep
   **Belongs to one**, and pick **Author** in **Target schema**.

![The schema builder with the Post content type open: its display name, machine name and fields with their types, and the relation panel of the author field.](images/schema-builder.png)

</div>
<div data-path="api">

Save this as `post-schema.json`:

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

`with_draft_publish` also turns on drafts and publishing, which step 5 explains.

</div>
</div>

[The data model](../concepts/data-model.md) lists every field type, every flag such as
`unique` and `default`, the validation rules and the other kinds of relation.

## 3. Preview the change

Ask what the new definition would change before you apply it.

<div class="paths">
<div data-path="console">

Choose **Preview DDL**. Each statement shows the SQL a save would run, and **Down SQL** shows
the SQL that undoes it. Choose **Close** when you have read them.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/schemas/post/preview-ddl \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @post-schema.json
```

The answer is `{"statements": [...]}`, each with a `description`, the `sql` an apply would
run and the `down_sql` that undoes it. The file also turns on drafts, so the API's preview adds
a status column.

</div>
</div>

For this change you see a column for `slug` and its index, and for the author a column, a step that clears links to authors that do not exist, a foreign
key and an index. Nothing changes until you apply.

## 4. Apply it

<div class="paths">
<div data-path="console">

Choose **Save**. The badge beside the name reads **Saved**. The Post has no drafts yet, and
step 5 turns them on.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @post-schema.json
```

The answer is `200` with the definition as stored. The Quickstart's first post, written before
drafts were on, is now `published`. Applying the same file again changes nothing.

</div>
</div>

The engine compares the new definition with the stored one and changes only what differs.
Existing entries keep their values.

Removing a field later does not delete its data right away. The removal waits until a super
admin applies it, so a mistaken edit loses nothing. [The data model](../concepts/data-model.md)
covers renames and removals.

## 5. Turn on drafts and publishing

With drafts on, each post carries a status: `draft`, `published` or `archived`. Only the API
turns drafts on: the schema builder in console v0.21.0 has no switch for them. If you followed
the API steps, `post-schema.json` turned them on in step 4, so go on to step 6.

From the console path, run two commands. They use the address of
[Run the engine and the console together](engine-and-console.md). An engine on its own takes
the same calls on port 3001. Sign in with your console account and keep the token:

```bash
curl -X POST http://localhost:8080/api/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "<your password>"}'

export TOKEN="<the token from the answer>"
```

Then save Post again with drafts turned on:

```bash
curl -X POST http://localhost:8080/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "post",
        "display_name": "Post",
        "with_draft_publish": true,
        "fields": [
          { "name": "title", "field_type": "text", "required": true },
          { "name": "body", "field_type": "rich_text" },
          { "name": "slug", "field_type": "text", "indexed": true },
          { "name": "author", "field_type": "relation", "relation_type": "belongs_to", "relation_to": "author" }
        ]
      }'
```

The answer is `200` with `"with_draft_publish": true`.

:::caution
Reload the schema builder before you save Post in it again. A schema builder that had Post open
before this call still holds the definition without drafts, and saving from it turns drafts
off and drops every post's status. Its **Preview DDL** shows that as a step that drops the
status column. After a reload, a save keeps drafts on.
:::

## 6. Write an author and a post

<div class="paths">
<div data-path="console">

1. Open **Content**, choose **Author** and then **New entry**.
2. Enter `Ada Lovelace` in **Name** and `Writes about engines.` in **Bio**, then choose
   **Create**.
3. Open **Content > Post** and choose **New entry**.
4. Enter a **Title**, a **Slug** such as `notes-on-the-engine` and some text in **Body**.
5. In **Author**, type part of the name and pick **Ada Lovelace**.
6. Choose **Create**.

A post created in the console starts as a draft. Open it and choose **Publish** to publish it.

</div>
<div data-path="api">

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
created through the Content API takes the `_status` its `data` names, such as `"_status": "draft"`,
and is published when it names none.

</div>
</div>

## 7. Take it back to draft, then publish it

<div class="paths">
<div data-path="console">

1. Open the post from **Content > Post** and choose **Unpublish**. Its status in the list
   reads `draft`.
2. The **Draft** and **Published** filters above the list show the entries in each status.
3. Open the post again and choose **Publish**.

</div>
<div data-path="api">

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

</div>
</div>

## 8. Read a post with its author

Reading a post together with its author is a Content API call, the one a site or an app
makes. The console shows the author's name in the **Author** column of the Post list instead.

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

`GET /api/v1/content/post/$POST_ID` reads one post by its `id`, and takes `populate` too.
Your account may read drafts, so a draft comes back as well. A caller who may not read drafts
gets `404` for one.

:::tip[A post with several authors]
Give `author` the `relation_type` `many_to_many` instead. The authors are then set with
`PUT /api/v1/content/post/{id}/relations/author` and a body of `{"ids": [...]}`, which
answers `204`, and read with `GET` on the same path. [The data model](../concepts/data-model.md)
describes each kind of relation and what deleting an entry does to it.
:::

## Next

- [Drafts and publishing](../guides/user/drafts-and-publishing.md): how editors work with
  drafts, revisions and scheduled publishing in the admin console.
- [Content history](../features/content-history.md): turn on revisions for the content types
  that need them.
- [Schema changes](../features/schema-changes.md): presets, renames and the history of every
  change.
- [Webhooks](../features/webhooks.md): tell another system when a post is published.
- [API endpoints](../reference/api-endpoints.md): paging, cursors and every other route that
  reads and writes entries.
