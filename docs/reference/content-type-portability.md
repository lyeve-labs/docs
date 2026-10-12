---
title: Moving content types between projects
description: "Export content types as a portable bundle, declare them in configuration, or import them from JSON Schema, OpenAPI, Strapi, Contentful and WordPress."
sidebar:
  order: 2
---

> **Included free** on every install.

This page moves content type **definitions**: names, fields, relations and switches. It moves
no entries. One bundle format serves three jobs, so an export from one instance is a file you
can read, review and replay on another.

| You want to | Use |
|---|---|
| Copy content types from one instance to another, such as staging to production | [Export](#export) on one, [import](#import) on the other |
| Keep the content model in your repository and apply it at startup | [Declaring content types in configuration](#declaring-content-types-in-configuration) |
| Start from the content model of a system you are leaving | [Importing from another system](#importing-from-another-system) |
| Move the entries as well | [Migrate existing content](../how-to/migrate-existing-content.md) |

To promote the model from a pipeline with these routes, see
[schema and flows as code](../how-to/schema-and-flows-as-code.md). What each key in a
definition means is on [the data model](../concepts/data-model.md) page.

## The bundle format

```yaml
version: 1
schemas:
  - name: author
    display_name: Author
    fields:
      - {name: name, field_type: text, required: true}
      - {name: email, field_type: email, unique: true}

  - name: post
    display_name: Post
    with_draft_publish: true
    fields:
      - {name: title, field_type: text, required: true, indexed: true}
      - {name: body, field_type: rich_text}
      - {name: author, field_type: relation, relation_to: author, relation_type: belongs_to}
```

The keys are the ones `POST /api/admin/schemas` takes, so anything you can build in the
**Schema builder** can be written here. A misspelled key is refused rather than ignored: a
bundle that spells `fields` as `field` answers `422` `parse bundle: json: unknown field "field"`,
where it would otherwise import a content type with no fields.

## Export

`GET /api/admin/schemas/export` writes every content type in your tenant, in dependency order,
so the file replays top to bottom. It takes `admin` or `super_admin`, or an
[admin token](../how-to/admin-tokens.md) with the `schemas:read` grant.

```bash
curl http://localhost:3001/api/admin/schemas/export \
  -H "Authorization: Bearer $TOKEN" -o schemas.yaml

curl "http://localhost:3001/api/admin/schemas/export?format=json" \
  -H "Authorization: Bearer $TOKEN" -o schemas.json
```

The file also carries `version: 1` and a `source` line naming the instance it came from. In
the console, open **Schema builder**, then the three-dot menu labeled **Import and export**,
and choose **Export as YAML** or **Export as JSON**.

## Import

`POST /api/admin/schemas/import` takes `super_admin` and a signed-in session, so an admin
token cannot call it. Importing is two steps: the first call reports what would change and
writes nothing, and the second applies it. Send YAML as `application/yaml` (`application/x-yaml`
and `text/yaml` work too), or JSON as `application/json`, up to 4 MiB:

```bash
curl -X POST http://localhost:3001/api/admin/schemas/import \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/yaml" \
  --data-binary @schemas.yaml
```

```json
{
  "applied": false,
  "plan": {
    "schemas": [
      { "name": "author", "action": "create", "ddl": ["CREATE TABLE IF NOT EXISTS ..."] },
      { "name": "post", "action": "create", "ddl": ["CREATE TABLE IF NOT EXISTS ...", "..."], "dependencies": ["author"] }
    ]
  }
}
```

Each content type gets an `action` of `create`, `update` or `unchanged`, with the statements it
would run in `ddl`. Read them before you apply: a change can drop columns as well as add them.
Then send the same file with `?apply=true`:

```bash
curl -X POST "http://localhost:3001/api/admin/schemas/import?apply=true" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/yaml" \
  --data-binary @schemas.yaml
```

The answer is the same shape with `"applied": true`. In the console, choose **Import** from the
same menu, pick **Where the definitions come from**, add the file or paste it, press **Check**
to see the plan, then **Import**.

### Order and references

A `belongs_to` field points at another table, so that table has to exist first. An import is
sorted by what each content type references, whatever order the file lists them in. When two
content types reference each other, the dry run names one of them in `second_pass`.

:::caution[Content types that reference each other]
Applying such a pair fails today when either content type is new: the answer is `409` and
nothing is created. Import both first without the relation fields that point at each other,
then send the full file again. With both tables in place, it applies.
:::

A reference to a content type that is neither in the file nor already in your tenant blocks the
whole import, and nothing is applied:

```json
{ "error": "import would fail: [post] reference schemas that are neither in the bundle nor in the target" }
```

The answer is `409`. A dry run of the same file lists the gap under `missing` for each content
type it affects. Import the missing content type first, or both together. A reference to one that already exists is fine, so a project can move
across a piece at a time.

## Declaring content types in configuration

Content types can live in `lyeve.yaml` beside your settings, so a whole project stands up from a
repository with no clicks in the console:

```yaml
$include: schemas/*.yaml

database:
  url: ${DATABASE_URL}
```

```yaml
# schemas/article.yaml
schemas:
  - name: article
    fields:
      - {name: title, field_type: text, required: true}
```

They are applied at startup, in the default tenant. A content type named in the files is created
or brought up to date. One that exists only in the database is left alone, because a file that
does not mention a content type is not an instruction to drop it and everything in it. Removing
a content type stays an explicit act through the
[schema routes](../features/schema-changes.md#delete-a-content-type).

If the declarations cannot be applied, the server does not start, so it never serves content
types other than the ones its configuration describes.
[Configuration](configuration.md) explains where the file is read from.

## Importing from another system

Add `from` to convert as you import. The dry run still comes first, so you see the result before
anything is written.

| System | `from` | What to send |
|---|---|---|
| LyEve bundle | `lyeve` (the default) | An export from this or another instance |
| JSON Schema | `json-schema` | An object schema, or a map of named schemas |
| OpenAPI | `openapi` | A document with `components.schemas`. Swagger 2 `definitions` work too |
| Strapi | `strapi` | One `schema.json`, or a list of them |
| Contentful | `contentful` | A space export, a list of content types, or one content type |
| WordPress | `wordpress` | An ACF field-group export, optionally with `/wp-json/wp/v2/types` |

```bash
curl -X POST "http://localhost:3001/api/admin/schemas/import?from=strapi" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --data-binary @strapi-article.json
```

### What conversion changes

A converted import carries two more fields. Read both before you apply.

`renamed` lists every name that had to change, because other systems allow names a content type
cannot have. `productName` becomes `product_name` and `my-field` becomes `my_field`:

```json
{
  "renamed": { "product.product_name": "productName -> product_name" },
  "notes": [{ "schema": "product", "field": "tags", "message": "kept as json: a list of values has no column type" }]
}
```

Check the renames before you move any entries, because the old names no longer exist.

`notes` lists everything that could not be carried across exactly. Conversion never drops a field
silently:

- A Strapi component or dynamic zone becomes a `json` field, and says so.
- A Contentful link that accepts several content types has no single target, so it becomes `json`.
- An enumeration becomes a text field, and its allowed values are not enforced.
- A password field is dropped rather than moved between systems.
- A relation becomes a relation only when the content type it points at is in the same import.
  Otherwise it becomes `json`, and the note names what it pointed at. Importing everything
  together gives a better result than one type at a time.

WordPress is the loosest fit, because WordPress has no schema to export. Each post type gets the
core fields (title, slug, content, excerpt, status, published date) plus whatever its ACF groups
define. Field groups attached to anything other than a post type, such as an options page, have
nowhere to land and are reported in `notes`.

## Errors

| Status | Message | Cause |
|---|---|---|
| `400` | `no content type definitions in the request body` | The body is empty. |
| `409` | `import would fail: [...] reference schemas that are neither in the bundle nor in the target` | A reference has nothing to point at. Nothing was applied. |
| `409` | `apply "<name>": ...` | Two content types in the file reference each other. Nothing was applied. See [Order and references](#order-and-references). |
| `413` | `content type definitions are too large` | The body is over 4 MiB. |
| `415` | `unsupported media type, expected application/json or multipart/form-data` | The `Content-Type` is not JSON or one of the YAML types. |
| `422` | The converter's reason, such as `unknown field "field"` or `unknown source "x"` | The file cannot be read, or `from` names no converter. |
