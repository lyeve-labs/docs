---
title: "9. Import, export and config sync"
description: "Load trips from a CSV file with a dry run first, save the column mapping as a template, export the guide on a schedule, and move its content types and settings to another engine."
sidebar:
  order: 9
covers:
  console:
    - "/admin/imports"
    - "/admin/imports/templates"
    - "/admin/data-export"
    - "/admin/settings/config-sync"
  api:
    - "POST /api/admin/imports/preview"
    - "POST /api/admin/imports/validate"
    - "POST /api/admin/imports"
    - "GET /api/admin/imports/{id}"
    - "POST /api/admin/imports/{id}/rollback"
    - "GET /api/admin/imports/templates"
    - "POST /api/admin/imports/templates"
    - "POST /api/admin/data-export/start"
    - "GET /api/admin/data-export/jobs"
    - "POST /api/admin/data-export/schedules"
    - "POST /api/admin/config-sync/export"
    - "POST /api/admin/config-sync/diff"
    - "POST /api/admin/config-sync/apply"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson trips arrive in bulk from a spreadsheet, the guide leaves as a file, and its
setup moves to a second engine. You need the `trip` content type from
[lesson 3](model-content.md), an `admin` or `super_admin` account, and a token in `TOKEN` for
the API steps. Imports take a signed-in session, so an admin token cannot run them.

Save this file as `trips.csv`:

```text
title,slug,starts_on,country
Alps by rail,alps-by-rail,2027-06-10,Switzerland
Scottish islands,scottish-islands,2027-07-02,United Kingdom
```

## Import trips from a file

> **Included free** on every install, every format included.

An import reads a file, matches its columns to the fields of one content type and saves each
row as an entry. A dry run comes first and writes nothing.

<div class="paths">
<div data-path="console">

1. Open **Insight > Imports** and choose **New import**. The drawer walks three steps:
   **Source**, **Map** and **Check**.
2. Under **Source**, keep **Upload a file**, add `trips.csv` under **File**, pick **Trip** in
   **Content type**, and keep **Format** at **Read it from the file**. Choose **Read the file**.
3. In **Map**, check the **Column in the file** for each **Field**. The console guesses them from
   the column names, and maps `title` to **Entry title** too.
4. Keep **When an entry already exists** at **Create only**, pick `slug` as the **Key field**,
   and choose **Dry run**.
5. **Check** shows **Rows in the file**, **Would import** and **Would be rejected**. Choose
   **Import 2 rows**.

</div>
<div data-path="api">

The API takes the file base64-encoded. Describe it, check it, then start it:

```bash
FD=$(base64 -w0 trips.csv)
MAP='[{"source_field": "title", "target_field": "title", "required": true},
      {"source_field": "slug", "target_field": "slug"},
      {"source_field": "starts_on", "target_field": "starts_on"},
      {"source_field": "country", "target_field": "country"}]'

curl -X POST http://localhost:3001/api/admin/imports/preview \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"source_name\": \"trips.csv\", \"file_data\": \"$FD\"}"

curl -X POST http://localhost:3001/api/admin/imports/validate \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"content_type\": \"trip\", \"source_name\": \"trips.csv\", \"file_data\": \"$FD\", \"field_mappings\": $MAP}"

curl -X POST http://localhost:3001/api/admin/imports \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"content_type\": \"trip\", \"source_name\": \"trips.csv\", \"source_format\": \"csv\", \"mode\": \"create\", \"file_data\": \"$FD\", \"field_mappings\": $MAP}"
```

The dry run answers `valid_rows` and `error_rows`. The start answers `201` with the job. Read
its progress with `GET /api/admin/imports/{id}`.

</div>
</div>

![A new import of trips.csv: the file and content type are chosen, the columns are mapped onto the fields, the dry run says 2 rows would import, and Import 2 rows starts it.](./media/import-and-export-import.mp4)

**History** lists every import with its outcome and how many rows it created, updated, skipped
and rejected, and updates until each finishes. A rejected row can be downloaded with its reason,
fixed and imported again. To undo an import, roll it back from its line, or call
`POST /api/admin/imports/{id}/rollback`, which deletes the entries it created.

![The Imports history after the import: trips.csv into Trip, with two rows created and none rejected.](./media/import-and-export-history.png)

[Bulk import](../features/bulk-import.md) lists the formats, the size limits and every
transform.

## Save the mapping as a template

> **Requires** a license with the `content-pro` feature. Applying and deleting a saved template
> stays free.

A template saves a column mapping with its transforms, key and mode, so the next file of the same
shape maps itself.

<div class="paths">
<div data-path="console">

1. Open the **Templates** tab of **Imports** and choose **New template**.
2. Enter a **Name**, pick **Trip** in **Content type**, and map each **Column in the file** to
   its **Field** the way the import did.
3. Pick `slug` as the **Key field** and save.
4. In the next **New import**, pick it in **Mapping template**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/imports/templates \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"Trips from a spreadsheet\", \"content_type\": \"trip\", \"upsert_key\": \"slug\", \"mode\": \"create\", \"field_mappings\": $MAP}"
```

`GET /api/admin/imports/templates` lists them, with `licensed` saying whether saving is allowed.

</div>
</div>

On a free install, **Templates** says saving templates and the date, split and lookup transforms
are not enabled on this engine, and saved templates still apply. With the license, **New
template** saves one.

![Templates on a free install: the notice that saving templates is not enabled on this engine.](./media/import-and-export-templates-free.png)

![Templates with a license: a template for trips is created with its columns, key field and mode.](./media/import-and-export-templates-licensed.mp4)

## Export the guide

> **Requires** a license with the `data-export` feature.

<div class="paths">
<div data-path="console">

1. Open **Insight > Data export** and choose **New export**.
2. Enter `trips-csv` in **Name**, pick **Trip** in **Content types** and CSV in **Format**.
   **Statuses** keeps the published and draft entries you want, and **Include the content type
   definitions** adds the model.
3. Choose **Start**. The export runs in the background, and the list updates until it finishes.
4. Download the file from its row under **Exports**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/data-export/start \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "trips-csv", "format": "csv", "filter": {"schemas": ["trip"]}}'

curl http://localhost:3001/api/admin/data-export/jobs \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![A new export of the trips as CSV is started, its row moves to finished, and the file is downloaded.](./media/import-and-export-export.mp4)

To run it every night, choose **New schedule** under **Export schedules**, give it a **Name**,
pick **When** or write a **Cron expression** in UTC, keep **Run on the timetable** on and choose
**Create**. Over the API, send the schedule to `POST /api/admin/data-export/schedules`.

On a free install, **Data export** says the feature is not enabled on this engine. With the
license, it shows the exports and the schedules.

![Data export on a free install: a notice that the feature is not enabled on this engine.](./media/import-and-export-data-export-free.png)

![Data export with a license: one finished export of trips and a nightly schedule.](./media/import-and-export-data-export-licensed.png)

[Data export](../features/data-export.md) covers every format, templates and sending a file
to S3.

## Move the setup to another engine

> **Requires** a license with the `config-sync` feature.

Config sync moves content types, flows, permission rules, webhooks and settings from one engine
to another as one bundle. It never moves entries. A super admin runs it, on both engines.

<div class="paths">
<div data-path="console">

1. On this engine, open **Settings > Config sync**. Under **Export this engine**, enter a
   **Passphrase** and choose **Export**, then **Download** the file. Anyone holding the file and
   the passphrase can read every secret in it.
2. On the second engine, open the same page. Under **Apply a bundle**, add the file in **Bundle
   file** and enter the **Passphrase**.
3. Choose **Compare** to see the plan: each item, the change and the fields it touches.
4. Choose **Dry run**. It writes nothing and lists any problem to fix first.
5. Choose **Apply**. The page says the engine now matches the bundle.

Tick **Delete items not in the bundle** only to remove flows, permission rules and webhooks that
exist on the second engine alone. Content types are never deleted.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/config-sync/export \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"passphrase": "a long passphrase from your secret store"}' \
  -o bundle.json
```

On the second engine, with a token of its own in `TOKEN`:

```bash
jq -n --slurpfile b bundle.json \
  '{passphrase: "a long passphrase from your secret store", bundle: $b[0]}' > plan.json

curl -X POST http://localhost:3001/api/admin/config-sync/diff \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @plan.json

jq '. + {dry_run: true}' plan.json > dry.json
curl -X POST http://localhost:3001/api/admin/config-sync/apply \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @dry.json

curl -X POST http://localhost:3001/api/admin/config-sync/apply \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @plan.json
```

Replace `localhost:3001` with the second engine's address when it runs elsewhere.

</div>
</div>

![A bundle is applied on a second engine: the file and passphrase are given, Compare lists the trip and trip_stop content types to create, the dry run passes, and Apply finishes.](./media/import-and-export-config-sync.mp4)

On a free install, **Config sync** says the feature is not enabled on this engine. With the
license, it shows the export and apply panels.

![Config sync on a free install: a notice that the feature is not enabled on this engine.](./media/import-and-export-config-sync-free.png)

![Config sync with a license: the Export this engine and Apply a bundle panels.](./media/import-and-export-config-sync-licensed.png)

[Promote configuration](../features/promote-configuration.md) lists what a bundle carries and
leaves out, how secrets travel, and how an apply runs.

## Next

- [10. API access and keys](api-access.md): give a site and a script their own credentials.
- [Bulk import](../features/bulk-import.md), [Data export](../features/data-export.md) and
  [Promote configuration](../features/promote-configuration.md): each in full.
- [Migrate existing content](../how-to/migrate-existing-content.md): bring content from another
  CMS.
