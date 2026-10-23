---
title: Migrate existing content
description: "Bring content into LyEve from files and other systems, move it between instances and tenants, or move the whole database to another server."
sidebar:
  order: 9
---

> **Included free** on every install. Moving the whole database requires a license with the `migration-toolkit` feature. See [pricing](https://lyeve.com/pricing/).

Moving content means moving two things: the **content types**, the shape, and the **entries**,
the content itself. LyEve has a tool for each kind of move, and none of them is a single
"import everything" button. Pick the row below that matches where your content lives today,
create or bring the content types first, then move the entries with a dry run before anything
is written.

## Which tool do I need?

| You have | Move the content types with | Move the entries with |
|---|---|---|
| A spreadsheet or any CSV, JSON, NDJSON or YAML file | The [Schema builder](../getting-started/first-content-type.md), by hand | [Bulk import](../features/bulk-import.md) |
| Strapi, Contentful, WordPress, a JSON Schema or an OpenAPI document | [Schema import with a converter](../reference/content-type-portability.md#importing-from-another-system) | Bulk import, from that system's export |
| Another LyEve instance, such as staging to production | [Schema export and import](../reference/content-type-portability.md) | A [data export](../features/data-export.md) file, read back by bulk import |
| Another tenant on the same instance or another one | [Schema export and import](../reference/content-type-portability.md), before the move | The [tenant migrate routes](#move-content-between-tenants) below |
| A tenant you want to copy whole, or keep a restorable copy of | Included | [Back up, restore and clone](backup-and-restore.md) |
| A whole database on another server | Included | The [database migrator](#move-the-whole-database) below |

Bulk import, the schema routes and the tenant migrate routes are free. Data export, tenant
backup and the database migrator need a license, as their pages say.

## Before you start

- For the API steps, you need a token in `TOKEN` for an `admin` or `super_admin`. The
  [quickstart](../getting-started/quickstart.md) shows how to get one. For the console steps,
  you sign in with the same role. Importing content types, and naming a tenant other than
  your own, take `super_admin`.
- The content types must exist before the entries arrive. Imports map records onto a content
  type you already have, and never create one.
- Take a [backup](backup-and-restore.md) of the target first.

## Move from another CMS

This brings a Strapi case study type and its entries across. The same steps work for any
source in the first two rows of the table. The type's Strapi definition is its
`schema.json`, saved here as `strapi-case-study.json`, with the singular name `case-study`
and the fields `title`, `views` and `body`. For the console steps, you are signed in to the
admin console as a super admin.

**1. Convert the content model and read the plan.** Nothing is written.

<div class="paths">
<div data-path="console">

1. Open **Schema builder**, choose **Import and export**, then **Import**.
2. Under **Where the definitions come from**, choose **Strapi content types**.
3. Drop `strapi-case-study.json` on **Definition file**, or paste it under
   **Or paste the definitions**, and choose **Check**.

**What importing does** shows `case_study` with **Creates**, the statements it runs, and
**Renamed to fit the engine's names**.

</div>
<div data-path="api">

```bash
curl -X POST "http://localhost:3001/api/admin/schemas/import?from=strapi" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --data-binary @strapi-case-study.json
```

The answer lists each content type with its `action` and statements, `renamed` for every
name that had to change, and `notes` for anything that could not be carried across exactly.

</div>
</div>

`case-study` becomes `case_study`, because a content type name takes no hyphen. Write down
the renames: your entry mappings use the new names. Strapi's draft and publish option is not
carried across, and the plan does not say so.

**2. Apply it.**

<div class="paths">
<div data-path="console">

Choose **Import** in the same drawer. The page confirms **Imported one content type**.

</div>
<div data-path="api">

Send the same file with `?apply=true`. The answer is the same plan with `"applied": true`.

</div>
</div>

**3. Check the entries against the new content type with a dry run.** Export them from the
old system as CSV or JSON. This example is a CSV file, `case-studies.csv`, with the header
`Title,Views` and one row.

<div class="paths">
<div data-path="console">

1. Open **Insight > Imports** and choose **New import**.
2. Under **Source**, keep **Upload a file**, drop `case-studies.csv` on **File**, choose
   `case_study` as the **Content type**, and choose **Read the file**.
3. Under **Map**, the `Title` column is matched to **Entry title** and `title`, and `Views` to
   `views`. Choose **Dry run**.

**Check** counts **Rows in the file**, **Would import** and **Would be rejected**, and reads
**Every row passes.** when none is rejected.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/imports/validate \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content_type": "case_study",
    "source_name": "case-studies.csv",
    "file_data": "VGl0bGUsVmlld3MKSGVsbG8sNDIK",
    "field_mappings": [
      {"source_field": "Title", "target_field": "title", "required": true, "transform": "trim"},
      {"source_field": "Views", "target_field": "views", "data_type": "integer"}
    ]
  }'
```

```json
{ "total_rows": 1, "valid_rows": 1, "error_rows": 0, "errors": [] }
```

`file_data` is your file, base64-encoded.

</div>
</div>

Fix the mapping until nothing is rejected, or accept the rows it rejects.

**4. Import the entries.**

<div class="paths">
<div data-path="console">

Choose **Import 1 row**. The page confirms that the import of `case-studies.csv` started.

</div>
<div data-path="api">

Post the same body to `POST /api/admin/imports`. The answer is `201` with a `pending` job.

</div>
</div>

The rows are written in the background. Imported entries are drafts unless you map
`entry.status`, and the Content API serves a draft only to a caller who may read drafts, as
the [data model](../concepts/data-model.md#statuses) page explains.

**5. Check the result.**

<div class="paths">
<div data-path="console">

**History** on **Insight > Imports** lists the import with its outcome, such as
**Every row imported**, and its created, updated, skipped and rejected counts. If it went
wrong, **Roll back the import** on its row deletes the entries it created.

</div>
<div data-path="api">

Read `GET /api/admin/imports/{id}` until it is `completed`, and list any rejected rows with
`GET /api/admin/imports/{id}/rows?status=errored`. If it went wrong,
`POST /api/admin/imports/{id}/rollback` deletes the entries the import created.

</div>
</div>

[Bulk import](../features/bulk-import.md) lists every mapping option, limit and error.

## Move entries between LyEve instances

1. Move the content types with the [schema export and import](../reference/content-type-portability.md).
2. On the source, start a [data export](../features/data-export.md) as `json`, `ndjson`, `yaml`
   or `csv`, and download it.
3. On the target, bulk import the file, one content type at a time. Each entry keeps its slug,
   title and status.

Set `"upsert_key": "entry.slug"` and `"mode": "upsert"`, and importing the same file again
updates the entries instead of duplicating them. A download link works as `source_url` only
when it is a full `http` or `https` address the target can reach.

## Move content between tenants

When both sides are LyEve tenants, the migrate routes export a tenant's content as a portable
payload and import it into the same or another tenant, renaming fields on the way. An admin
works on their own tenant. Naming another tenant in `tenant_id` takes `super_admin`. These
routes are API only on v0.21.0.

The content types have to exist in the target tenant first. The import maps records onto
them and never creates one, and a missing type answers `404` with `import failed`.

1. Export one content type as JSON Lines:

   ```bash
   curl -X POST http://localhost:3001/api/admin/migrate/export \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"schemas": ["product"], "format": "jsonl"}' \
     -o export.jsonl
   ```

   An export sends its status before the body streams, so a failure part way cannot change the
   status code. Check the `X-Export-Status` trailer and the final `{"_type": "result"}` line.

2. Dry-run the import into the `staging` tenant, renaming `title` to `name`:

   ```bash
   curl -X POST http://localhost:3001/api/admin/migrate/validate \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d "$(jq -Rs '{
           tenant_id: "staging",
           field_mapping: [
             {source_schema: "product", target_schema: "product",
              fields: [{source_field: "title", target_field: "name"}]}
           ],
           data: .
         }' export.jsonl)"
   ```

   Every content type in the file needs an entry in `field_mapping`, even one that renames
   nothing. A record whose content type has none is skipped and named in `validation_errors`.

3. Post the same body to `POST /api/admin/migrate/import` with `"dry_run": false` to write it.
   The answer counts:
   - `records_imported`.
   - `records_present`: rows the tenant already had under the same id, left as they were.
   - `records_reidentified`: rows written under a new id because the id is taken elsewhere on
     the instance.
   - `records_skipped`, which counts the export's closing result line as one. That line also
     shows in `validation_errors` as `no mapping for source schema ""`, which you can ignore.

   Running an import again is safe.

The import body is limited to 1 MiB, so split a large tenant by content type.

`POST /api/admin/migrate/sqldump/export` writes a portable dump instead, the same whatever
database wrote it, and `"include_ddl": true` adds each table's definition to it.
`POST /api/admin/migrate/sqldump/import` loads it on PostgreSQL, MySQL or SQL Server, with
the same rules as the JSON Lines import: the content types must already exist in the target
tenant, because the definitions in the dump are not applied, and every content type needs an
entry in `field_mapping` or its records are skipped. Its `data` holds at most 65,535
characters, so the dump suits a small tenant.

## Move the whole database

> **Requires** a license with the `migration-toolkit` feature. See [pricing](https://lyeve.com/pricing/).

The database migrator copies the database under an install from one connection string to
another, in any pairing of PostgreSQL, MySQL and SQL Server. It copies tables and rows, not
constraints, keys or indexes. `db` is the only job `type`, and the feature is in beta. It is
API only on v0.21.0.

1. Read the compatibility report. Nothing is touched:

   ```bash
   curl -X POST http://localhost:3001/api/admin/migration/compatibility \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"type": "db", "source_dsn": "postgres://user:pass@source-db.example.com:5432/app", "target_dialect": "mysql"}'
   ```

   It lists the type each column maps to, the incompatibilities that block the move, and
   warnings. A stock install read for MySQL reports `"compatible": false`, because its
   PostgreSQL array columns have no MySQL type and are carried as text, and it warns that
   every time zone is dropped and values are stored in UTC.

2. Start the copy:

   ```bash
   curl -X POST http://localhost:3001/api/admin/migration/start \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"type": "db", "source_dsn": "postgres://user:pass@source-db.example.com:5432/app", "target_dsn": "postgres://user:pass@target-db.example.com:5432/app", "target_dialect": "postgres", "batch_size": 5000}'
   ```

   The answer is `202` with the job. `batch_size` is 1 to 100,000, and 1,000 by default.

3. Follow `GET /api/admin/migration/jobs/{id}` for `progress`, `migrated_rows` and
   `failed_rows`. `POST .../cancel` stops a pending or running job, and `POST .../rollback`
   drops only the tables the job created, answering
   `{"dropped_tables": 262, "status": "rolled_back"}` with its own count.

Connection strings that point at loopback, private, link-local or cloud metadata addresses are
refused with `400` `host is not an allowed migration target`, and so is a host name that does
not resolve. To reach a private network, your
operator lists it in `MIGRATION_ALLOWED_PRIVATE_NETWORKS` before the engine starts. With the
[web application firewall](../features/web-application-firewall.md) on, a connection string
that names an internal IP address is refused with `403` and `waf_blocked` before the migrator
reads it, so name the database by its host name. Passwords in connection strings show as
`***` in every answer and never come back over the API.

## Next

- [Bulk import](../features/bulk-import.md): every mapping option, limit and error.
- [Content type portability](../reference/content-type-portability.md): export, import and
  convert content types.
- [Back up and restore](backup-and-restore.md): copy or clone a tenant whole.
- [Tenants](../features/tenants.md): how tenants are kept apart.
