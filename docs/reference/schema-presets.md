---
title: Schema Presets
description: "Create a ready-made set of content types in one call, save presets of your own, and add the flow templates that act on comments, forms and SEO metadata."
sidebar:
  order: 3
---

> **Included free** on every install. Saving a preset of your own needs a license with the `schema-pro` feature. See [pricing](https://lyeve.com/pricing).

A preset creates a small set of content types in one call, with their fields, relations and
validation already written. It is a starting point you own. Once created, the content types
are ordinary ones that you edit, rename and delete like any other, and nothing ties them back
to the preset. Three presets are built in and free on every install. With Schema Pro you can save your own
beside them.

## Pick a preset

| Preset | Creates | Good for |
|---|---|---|
| `comments` | `comments`, `comment_votes` | Threaded comments on any entry, with votes and a moderation status |
| `forms` | `forms`, `form_submissions` | Contact and sign-up forms whose fields you define as data, and the answers they collect |
| `seo` | `seo_metadata` | Page titles, descriptions, Open Graph and structured data for any entry |

To change a content type after it exists, see [Schema changes](../features/schema-changes.md).
To bring definitions from another instance or another CMS, see
[content type portability](content-type-portability.md).

## Apply a preset

In the admin console, open **Schema builder**, choose **Presets**, and press **Create** on the
preset you want. Over the Admin API, with a token in `TOKEN` for an `admin` or `super_admin`
(the [quickstart](../getting-started/quickstart.md) shows how to get one):

```bash
curl -X POST http://localhost:3001/api/admin/schemas/presets/forms \
  -H "Authorization: Bearer $TOKEN"
```

```json
{ "created": ["forms", "form_submissions"] }
```

The answer is `201`. The content types are created in the order the preset lists them,
exactly as if you had posted each one to `POST /api/admin/schemas`.

- **A name you already use.** If any content type the preset creates already exists, the
  answer is `409` `a schema the preset creates already exists` and nothing is created. A
  preset never merges into an existing content type, because nothing can know whether the
  fields it would add are wanted. Rename or delete yours first, or copy the fields you want
  from the definitions below.
- **A failure part way.** Some databases cannot roll back a table change, so a preset that
  fails after its first content type keeps the ones it created. Check which exist before you
  try again.

`GET /api/admin/schemas/presets` answers `{"presets": [...], "can_save": true}` with every
built-in and saved preset: its `id`, `name`, `description`, `source` (`builtin` or `saved`)
and the full definition of each content type it creates. A saved one also carries
`updated_at`, and `source_url` when it was read from a URL. `can_save` says whether saving a
preset of your own would be accepted on this install.

## Save a preset of your own

A preset of your own carries your team's starting model, such as an events calendar, so every
new tenant or project starts from the same definitions. Saving one needs a license with the
`schema-pro` feature, and without it `POST /api/admin/schemas/presets` answers `402`. Listing,
applying and deleting a preset you already saved stay free, so a lapsed license keeps every
preset you saved and only stops you saving more.

In the console, open **Presets** and use **Save a preset from a URL or a file**. You can give
it a URL, upload a file or paste the document. Without Schema Pro the console says that saving
presets is part of it, and the built-in presets stay. Over the API,
`POST /api/admin/schemas/presets` takes either a `document` or a `url` to read one from, never
both:

```bash
curl -X POST http://localhost:3001/api/admin/schemas/presets \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"document": {"id": "events", "name": "Events", "description": "Events at a venue.", "schemas": [{"name": "venue", "fields": [{"name": "name", "field_type": "text", "required": true}]}, {"name": "event", "fields": [{"name": "title", "field_type": "text", "required": true}, {"name": "venue", "field_type": "relation", "relation_to": "venue", "relation_type": "belongs_to"}]}]}}'
```

The answer is `201` with the saved preset. Apply it with
`POST /api/admin/schemas/presets/events`, like a built-in one.

- The document has the shape of the built-in presets below, as JSON or YAML. A YAML document
  goes in `document` as a JSON string.
- The `id` is a lower-case slug, and cannot be the id of a built-in preset.
- Every content type needs at least one field, and a relation may only point at its own
  content type or at one listed before it.
- A document is at most 1 MiB. A URL must be `http` or `https`. One that resolves to a
  private, loopback or link-local address is never fetched, at every redirect too, and answers
  `502` like a URL that does not serve a preset.
- Saving the same id again replaces it, which is how a preset read from a URL is refreshed.

`DELETE /api/admin/schemas/presets/{id}` removes a saved preset and answers `204`. The content
types it already created stay.

<details>
<summary>Errors from the preset routes</summary>

| Status | Message |
|---|---|
| `400` | `invalid JSON` |
| `402` | `payment_required`, when saving a preset without `schema-pro` |
| `404` | `preset not found` |
| `409` | `a schema the preset creates already exists` |
| `409` | `a built-in preset already uses that id` |
| `409` | `a built-in preset cannot be deleted` |
| `422` | `a url or a document is required`, `send a url or a document, not both`, `the URL must be http or https`, or what is wrong with the document |
| `422` | `invalid schema in preset` |
| `502` | `the URL did not answer with a preset document` |

</details>

## The `comments` preset

A comment names the entry it belongs to by content type and id, so one `comments` content type
serves every kind of entry.

| Field | Purpose |
|-------|---------|
| `target_schema` | The content type of the entry the comment is on. |
| `target_id` | The id of that entry. |
| `parent` | The comment this one replies to, a `belongs_to` on `comments`. Read back as `parent_id`. Empty for a top-level comment. |
| `author_name` | Display name of the author. |
| `author_email` | Address of the author. Every read returns it like any other field, so hide it from public readers with a `field_mask` in an [access rule](../features/access-rules.md). |
| `body` | The comment text. |
| `status` | One of `pending`, `approved`, `spam` or `rejected`, checked on every write. Defaults to `pending`. |

`comment_votes`:

| Field | Purpose |
|-------|---------|
| `comment` | The comment voted on, a `belongs_to` on `comments`. Read back as `comment_id`. |
| `voter_key` | Whatever identifies the voter to you: a user id, a session id, a hash. |
| `value` | The vote, such as `1` or `-1`. |

A site lists approved comments through the Content API with
`?filters[target_schema]=post&filters[target_id]=<id>&filters[status]=approved`, and posts a
comment as a content write, which lands as `pending`. Moderation is a change of `status`,
which the `comment-moderation` flow template can make for you.

<details>
<summary>The comments preset as the server ships it</summary>

```json
{
  "id": "comments",
  "name": "Comments",
  "description": "Threaded comments on any content type, with votes. A comment names the schema and row it belongs to, answers to a parent comment through the parent relation (its column is parent_id), and starts pending until a moderation flow approves it. Moderation, spam scoring, mention parsing and vote de-duplication are not part of the preset; a flow on comments.after_create carries what the tables cannot.",
  "schemas": [
    {
      "name": "comments",
      "display_name": "Comments",
      "with_created_at": true,
      "with_updated_at": true,
      "fields": [
        {"name": "target_schema", "field_type": "text", "required": true, "indexed": true},
        {"name": "target_id", "field_type": "text", "required": true, "indexed": true},
        {"name": "parent", "field_type": "relation", "relation_to": "comments", "relation_type": "belongs_to"},
        {"name": "author_name", "field_type": "text"},
        {"name": "author_email", "field_type": "email"},
        {"name": "body", "field_type": "text", "required": true},
        {
          "name": "status",
          "field_type": "text",
          "required": true,
          "indexed": true,
          "default": "pending",
          "validation": [{"rule": "enum", "params": {"values": ["pending", "approved", "spam", "rejected"]}}]
        }
      ]
    },
    {
      "name": "comment_votes",
      "display_name": "Comment votes",
      "with_created_at": true,
      "fields": [
        {"name": "comment", "field_type": "relation", "relation_to": "comments", "relation_type": "belongs_to", "required": true},
        {"name": "voter_key", "field_type": "text", "required": true, "indexed": true},
        {"name": "value", "field_type": "number", "required": true}
      ]
    }
  ]
}
```

</details>

## The `forms` preset

A form is data: its field list is JSON that your page renders, so you add a form without a
deploy.

| Field | Purpose |
|-------|---------|
| `name` | The form's display name. |
| `slug` | The name a site posts to. Unique. |
| `fields` | The form's field list, as JSON: what the page renders and what a submission carries. |
| `notify_email` | Where a submission is announced. The `form-submitted` flow template reads it. |
| `active` | Whether the form takes submissions. Defaults to `true`. |

`form_submissions`:

| Field | Purpose |
|-------|---------|
| `form` | The form submitted, a `belongs_to` on `forms`. Read back as `form_id`. |
| `data` | The submitted values, as JSON. |
| `status` | One of `new`, `read` or `spam`, checked on every write. Defaults to `new`. |

A page reads the form through the Content API, renders `fields`, and posts the answers as a
`form_submissions` entry. Reading and triage are content edits. A file a form collects goes
through [Media](../features/media.md), and the submission carries its link.

<details>
<summary>The forms preset as the server ships it</summary>

```json
{
  "id": "forms",
  "name": "Forms",
  "description": "Form definitions and the submissions they collect. A form carries its field layout as JSON and a notification address; a submission holds the posted data as JSON and starts as new. Delivery of the notification, uploads, honeypot scoring and a submit endpoint of its own are not part of the preset; a flow on form_submissions.after_create sends the email and calls out.",
  "schemas": [
    {
      "name": "forms",
      "display_name": "Forms",
      "with_created_at": true,
      "with_updated_at": true,
      "fields": [
        {"name": "name", "field_type": "text", "required": true},
        {"name": "slug", "field_type": "text", "required": true, "unique": true},
        {"name": "fields", "field_type": "json", "required": true},
        {"name": "notify_email", "field_type": "email"},
        {"name": "active", "field_type": "boolean", "default": true}
      ]
    },
    {
      "name": "form_submissions",
      "display_name": "Form submissions",
      "with_created_at": true,
      "fields": [
        {"name": "form", "field_type": "relation", "relation_to": "forms", "relation_type": "belongs_to", "required": true},
        {"name": "data", "field_type": "json", "required": true},
        {
          "name": "status",
          "field_type": "text",
          "required": true,
          "indexed": true,
          "default": "new",
          "validation": [{"rule": "enum", "params": {"values": ["new", "read", "spam"]}}]
        }
      ]
    }
  ]
}
```

</details>

## The `seo` preset

One content type holds the metadata a page carries in its `<head>`.

| Field | Purpose |
|-------|---------|
| `entry_schema` | The content type of the entry the metadata is for. |
| `entry_id` | The id of that entry. |
| `title` | The page title. |
| `description` | The meta description. |
| `canonical_url` | The canonical URL. |
| `robots` | The robots directive. Defaults to `index,follow`. |
| `og_title`, `og_description`, `og_image` | The Open Graph title, description and image. |
| `twitter_card` | The Twitter Card type. Defaults to `summary_large_image`. |
| `structured_data` | The schema.org JSON-LD payload, as JSON. |

A front end reads a page's metadata through the Content API with a filter on `entry_schema`
and `entry_id`, and writes the tags.

<details>
<summary>The seo preset as the server ships it</summary>

```json
{
  "id": "seo",
  "name": "SEO metadata",
  "description": "Search and social metadata for any content entry. A row names the schema and entry it describes and carries the title, description and canonical URL a page head needs, the robots directive, the Open Graph and Twitter card fields, and structured data as JSON-LD. A front end reads it through the content API in the locale it renders, and the sitemap and robots.txt documents that follow the content are flow templates rather than part of the preset.",
  "schemas": [
    {
      "name": "seo_metadata",
      "display_name": "SEO metadata",
      "with_created_at": true,
      "with_updated_at": true,
      "fields": [
        {"name": "entry_schema", "field_type": "text", "required": true, "indexed": true},
        {"name": "entry_id", "field_type": "text", "required": true, "indexed": true},
        {"name": "title", "field_type": "text"},
        {"name": "description", "field_type": "text"},
        {"name": "canonical_url", "field_type": "text"},
        {"name": "robots", "field_type": "text", "default": "index,follow"},
        {"name": "og_title", "field_type": "text"},
        {"name": "og_description", "field_type": "text"},
        {"name": "og_image", "field_type": "text"},
        {"name": "twitter_card", "field_type": "text", "default": "summary_large_image"},
        {"name": "structured_data", "field_type": "json"}
      ]
    }
  ]
}
```

</details>

## Flow templates that act on presets

A content type stores data. It does not act on it. Four [flow templates](../how-to/flow-templates.md)
carry the behavior, each an ordinary flow you own and edit:

| Template | What it does | Needs |
|---|---|---|
| `comment-moderation` | On each new `pending` comment, posts it to an HTTP datasource named `moderation` and sets the status to `spam` or `approved` from the answer. An answer with no verdict leaves it `pending` for an editor. With the [AI](../features/ai.md) feature licensed, an `ai.classify` node can stand in for the request. | `flow-pro` |
| `form-submitted` | On each new submission, emails it to the form's `notify_email`, and posts it to an HTTP datasource named `form_webhook` when the `form_webhook` variable is `on`. | `flow-pro` and a configured [email](../features/email.md) sender |
| `seo-sitemap` | A public `GET` that renders `sitemap.xml` from the `pages` entries whose `status` is `published`, using their `slug` and `updated_at`. Needs a `site_url` variable. | Free |
| `seo-robots` | A public `GET` that renders `robots.txt` from a `robots_rules` content type, one line per entry, with a `Sitemap` line from the `sitemap_url` variable. | Free |

The two SEO templates answer at the flow's public URL, `/api/v1/flows/p/<flow_id>`. Point
`/sitemap.xml` and `/robots.txt` at them with a rewrite at your edge. Any other flow that
reacts to a comment or a submission starts from an event trigger on the `comments` or
`form_submissions` content type. See [Flows](../features/flows.md).

## What a preset leaves to you

- **One vote per voter.** `comment_votes` stores what it is given, so a second vote under the
  same `voter_key` is a second entry. Refuse it in a flow, or check before you write.
- **Mentions.** A comment body is text. A flow can parse `@name` mentions out of it.
- **Spam checks on forms.** A honeypot or a captcha check is a condition in a flow, or
  [CAPTCHA](../features/captcha.md) in front of the page.
- **Export.** List submissions through the Content API, or use
  [Data export](../features/data-export.md).

## Related

- [Data model](../concepts/data-model.md): fields, relations and statuses.
- [Schema changes](../features/schema-changes.md): change a content type after a preset
  creates it.
- [Flow templates](../how-to/flow-templates.md): the templates above, in full.
