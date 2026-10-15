---
title: Back up and restore tenants
description: "Export, validate, import, restore and clone one tenant, and run backup jobs that write encrypted archives to S3."
sidebar:
  order: 7
---

> **Requires** a license with the `tenant-backup` feature. See [pricing](https://lyeve.com/pricing/).

This page takes one [tenant](../features/tenants.md) out of the instance and
puts it back: as a portable SQL file you download, as an encrypted archive in an
S3 bucket, or as a clone into a new tenant on the same instance. A file taken on
PostgreSQL loads on MySQL or SQL Server, and the other way round. These routes
cover one tenant's content types and entries. Back up the whole database with
your database provider's own tooling.

## Before you start

- Every route needs the `admin` or `super_admin` role. An admin works on their
  own tenant. Naming another tenant needs `super_admin` and otherwise answers
  `403`, for example `cross-tenant export requires super_admin`.
- Without `tenant-backup` every route here answers `402`. If you set
  `LYEVE_PLUGINS`, include `multitenant`. See
  [Licensing and tiers](../concepts/licensing-and-tiers.md).
- `import` and `restore` replace a tenant's data, so they take a signed-in
  session. An admin token or an API key cannot call them.
- `import` and `validate` carry the dump inside a JSON body, so the instance's
  JSON limit applies first: `MAX_JSON_BODY_BYTES`, 1 MiB by default, answers
  `413` above it. To load a larger dump, raise it (it applies to every JSON
  route) and keep it within `MAX_BODY_BYTES`, 10 MiB by default. The routes'
  own ceilings are 500 MiB for `import` and 50 MiB for `validate`.
- With the [web application firewall](../features/web-application-firewall.md)
  running, its SQL injection rules refuse `validate` and `import` with `403`
  and `request blocked by WAF`, because the body is SQL. Add an exception for
  these paths before you load a dump.
- The feature is in beta.

## Export a tenant to a file

1. Download the tenant as one SQL document, table definitions and rows:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-backup/export \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"include_data": true, "target_dialect": "postgres"}' \
     -o tenant-backup.sql
   ```

   The answer is `application/sql` with an attachment name such as
   `tenant-acme-backup-20261002-091515.sql`. The file opens with comments
   naming the tenant and both dialects, and ends with a summary line such as
   `-- Export summary: 2 tables, 2 rows, elapsed 19.29ms`.

2. Check what it holds before you rely on it:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-backup/validate \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d "$(jq -Rs '{sql: .}' tenant-backup.sql)"
   ```

   Nothing is written. The answer counts `statements_executed` and
   `rows_inserted`, and lists any `errors`, each with its `statement_no`, a
   `message` and a `preview` of the statement.

| Export field | Meaning |
|---|---|
| `tenant_id` | The tenant to export. Empty means your own. Another tenant needs `super_admin`. |
| `schemas` | Content types to include. Empty means all. |
| `include_data` | `true` adds the rows. Otherwise the file holds only the table definitions. |
| `target_dialect` | `postgres`, `mysql`, `mssql`, or empty for the source database's own. Anything else answers `400` with `invalid target_dialect`. |

`GET /api/admin/tenant-backup/tables` lists the tenant's tables with their row
and column counts.

## Load a dump into a tenant

Load into a staging tenant first and check it there. Never truncate a
production tenant on a dump you have not loaded somewhere else.

1. Dry-run the load. `dry_run` parses and counts, and writes nothing:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-backup/import \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d "$(jq -Rs '{tenant_id: "acme_restore", dry_run: true, sql: .}' tenant-backup.sql)"
   ```

2. Run it for real by dropping `dry_run`. Add `truncate` to empty each target
   table first, or `skip_existing` to leave any table that already has rows
   alone.

   The answer reports `statements_executed`, `statements_skipped`,
   `rows_inserted`, `rows_foreign_tenant` (rows in the file that belong to
   another tenant, which are not loaded), any `errors`, and `elapsed`.

| Import field | Effect |
|---|---|
| `tenant_id` | The target tenant. Empty means your own. |
| `sql` | The dump. Required, or the call answers `400` with `sql is required`. |
| `dry_run` | Parse and count only. |
| `truncate` | Empty each target table **before** loading. This deletes the existing data. |
| `skip_existing` | Leave any table that already has rows untouched. |

## Back up to S3

A backup job exports a tenant, checksums it, encrypts it and uploads it to an S3
bucket each time it runs. When `REPLICATION_TARGETS` is set, the archive is also
copied to each enabled target.

1. Create the job:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-backup/jobs \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name": "nightly", "cron_expr": "0 3 * * *", "include_data": true,
          "retention_days": 30, "s3_bucket": "lyeve-backups", "s3_prefix": "backups"}'
   ```

   The answer is `201` with the job and its `id`. `name` and `s3_bucket` are
   required. When the instance sets `STORAGE_S3_BUCKET`, a job may write only to
   that bucket or one listed in `STORAGE_S3_BACKUP_ALLOWED_BUCKETS`. Any other
   answers `400` with `s3_bucket "<name>" is not in the allowed backup buckets
   list`.

2. Run it:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-backup/jobs/$JOB_ID/trigger \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer is `202` with a `run_id`, and the backup runs after it, for up to
   30 minutes.

3. Follow the run with `GET /api/admin/tenant-backup/runs/{id}`. Its `status`
   moves from `running` to `completed` or `failed`. A failed run's message
   points at the server log, which has the detail.

4. Check the archive it wrote:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-backup/verify \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"archive_id": "'"$ARCHIVE_ID"'"}'
   ```

   The answer carries `checksum_ok` and `sql_valid`. List archives with
   `GET /api/admin/tenant-backup/archives`.

:::caution[Nothing runs a job on its schedule]
A job stores `cron_expr`, but the instance does not run it on that schedule.
Call the `trigger` route from your own scheduler, such as a Kubernetes CronJob,
at the cadence you want. Archives past `retention_days` are deleted only when
you call `POST /api/admin/tenant-backup/retention/cleanup`. The job, run and
archive records are kept.
:::

Archives are encrypted with a key derived for each tenant from the instance's
`ENCRYPTION_KEY`, or by AWS KMS when `KMS_PROVIDER=aws`. Keep that key safe:
an archive cannot be restored without it.

With `KMS_PROVIDER=aws`, each archive is encrypted under a fresh data key that
AWS KMS generates and wraps with `KMS_KEY_ID`. Restoring the archive asks KMS
to unwrap that key. Credentials come from the standard AWS chain: the
environment, web identity, the shared credentials and config files, and
container or instance credentials. Grant the engine's role
`kms:GenerateDataKey` and `kms:Decrypt` on the key.

## Restore from an archive

1. Dry-run the restore into a staging tenant:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-backup/restore \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"archive_id": "'"$ARCHIVE_ID"'", "target_tenant": "acme_restore", "dry_run": true}'
   ```

2. Drop `dry_run` to restore. Add `"truncate": true` to empty the target tables
   first. `target_tenant` empty means your own, and another tenant needs
   `super_admin`. The answer has the same counts as an import.

## Test restores

A restore test takes the tenant's newest archive that is at least an hour old,
checks its checksum and SQL, and loads it into a test tenant, then records
`passed` or `failed` with the tables and rows it imported.

```bash
curl -X POST http://localhost:3001/api/admin/tenant-backup/restore-tests/schedules \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"schedule_cron": "0 4 * * 1", "test_tenant": "acme_restore_test", "timeout_sec": 600, "enabled": true}'
```

Run it with `POST .../restore-tests/schedules/{id}/trigger`, which answers with
the result once the test finishes, and read past results at
`GET .../restore-tests/results`. `timeout_sec` defaults to 600.

:::danger[A restore test empties its test tenant]
The test truncates every table it loads in `test_tenant`. Point it at a tenant
that holds nothing you need, never at a live one. Like a backup job, a schedule
stores `schedule_cron` but runs only when you call `trigger`.
:::

## Clone a tenant

A clone copies a source tenant's content types and entries into a new tenant on
the same instance, without a file. Use it to seed a staging tenant from
production or to stand up a demo. It copies content only, not what other
features keep for the tenant.

A clone creates its target, so it needs `super_admin`, and anyone else answers
`403` with `only super_admin may provision new tenants`. A real run also needs
`multitenant-provisioning` and room under the tenant ceiling, and otherwise
answers `402`.

1. Validate:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-clone/validate \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"source_tenant_id": "acme", "target_tenant_id": "acme_staging"}'
   ```

   ```json
   {"valid": true, "issues": null}
   ```

   Validation checks that the source exists and has content types, and that the
   target slug is valid and not taken. Each issue carries a `severity` of
   `error` or `warning`, a `category` and a `message`. An `error` blocks the
   clone.

2. Dry-run it. Every check runs and nothing is written:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenant-clone/clone \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"source_tenant_id": "acme", "target_tenant_id": "acme_staging",
          "target_name": "Acme Staging", "dry_run": true}'
   ```

   ```json
   {"source_tenant": "acme", "target_tenant": "acme_staging", "dry_run": true,
    "schemas_cloned": 1, "rows_cloned": 2, "rows_skipped": 0, "elapsed": "1.49ms"}
   ```

3. Run it with `"dry_run": false` and `curl -N`. The answer streams
   Server-Sent Events, `phase`, `schema` and `row` while it works, and ends with
   `event: done` carrying the same counts, or `event: error`.

Every copied entry gets a new id, and relations point at the copies. If a live
clone fails after creating the target, the target is removed so you can retry
the same slug.

| Clone field | Meaning |
|---|---|
| `source_tenant_id`, `target_tenant_id`, `target_name` | Required. |
| `target_plan` | The new tenant's plan. Defaults to `free`. |
| `dry_run` | `true` runs every check and answers JSON. |
| `schemas` | Content types to copy. Empty means all. |
| `skip_content` | Copy the content types without their entries. |
| `field_mapping` | A list of `{"source_schema", "target_schema", "fields": [{"source_field", "target_field"}]}` that renames fields on the way. |

## Settings

| Variable | What it does | Default |
|---|---|---|
| `STORAGE_S3_BUCKET` | The bucket backups go to, and the only one a job may name unless the next setting adds more. | unset |
| `STORAGE_S3_BACKUP_ALLOWED_BUCKETS` | Comma-separated extra buckets a job may write to. | none |
| `STORAGE_S3_REGION`, `STORAGE_S3_ENDPOINT` | Region and a custom endpoint, for an S3-compatible service. | unset |
| `STORAGE_S3_KEY`, `STORAGE_S3_SECRET` | S3 credentials. | unset |
| `STORAGE_S3_FORCE_PATH_STYLE` | Path-style S3 addressing. | `false` |
| `KMS_PROVIDER` | `aws` for AWS KMS, or empty for local AES-256-GCM encryption. | local |
| `KMS_REGION`, `KMS_KEY_ID`, `KMS_ENDPOINT` | AWS KMS settings. | unset |
| `REPLICATION_TARGETS` | A JSON array of copies to make of every archive, each `{"region", "bucket", "prefix", "endpoint", "access_key", "secret_key", "path_style", "use_ssl", "enabled"}`. Only enabled targets are used. | none |

[Object storage](../features/object-storage.md) covers the S3 settings in full.

## Routes

<details>
<summary>Backup, restore and clone</summary>

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/admin/tenant-backup/export` | Download a tenant as SQL. |
| `POST` | `/api/admin/tenant-backup/validate` | Parse a dump and count it. Nothing is written. |
| `POST` | `/api/admin/tenant-backup/import` | Load a dump into a tenant. Session only. |
| `GET` | `/api/admin/tenant-backup/tables` | The tenant's tables with row and column counts. |
| `GET` | `/api/admin/tenant-backup/progress` | Follow a running export or import as Server-Sent Events, with `?op_id=`. |
| `GET`, `POST` | `/api/admin/tenant-backup/jobs` | List or create backup jobs. |
| `GET`, `PUT`, `DELETE` | `/api/admin/tenant-backup/jobs/{id}` | Read, change or delete a job. |
| `POST` | `/api/admin/tenant-backup/jobs/{id}/trigger` | Run a job now. |
| `GET` | `/api/admin/tenant-backup/runs`, `/runs/{id}` | List runs or read one. |
| `GET` | `/api/admin/tenant-backup/archives`, `/archives/{id}` | List archives or read one. |
| `GET` | `/api/admin/tenant-backup/archives/{id}/download` | Download an archive. |
| `DELETE` | `/api/admin/tenant-backup/archives/{id}` | Delete an archive. |
| `POST` | `/api/admin/tenant-backup/restore` | Restore an archive into a tenant. Session only. |
| `POST` | `/api/admin/tenant-backup/verify` | Check an archive's checksum and SQL. |
| `POST` | `/api/admin/tenant-backup/retention/cleanup` | Delete archives past their retention now. |
| `GET` | `/api/admin/tenant-backup/replication/status` | Replication targets. Super admin only. |
| `GET`, `POST` | `/api/admin/tenant-backup/restore-tests/schedules` | List or create restore-test schedules. |
| `GET`, `PUT`, `DELETE` | `/api/admin/tenant-backup/restore-tests/schedules/{id}` | Read, change or delete one. |
| `POST` | `/api/admin/tenant-backup/restore-tests/schedules/{id}/trigger` | Run a restore test now. |
| `GET` | `/api/admin/tenant-backup/restore-tests/results` | Restore-test results. |
| `POST` | `/api/admin/tenant-clone/validate` | Check a clone. |
| `POST` | `/api/admin/tenant-clone/clone` | Clone a tenant. |

</details>

Archives and exports contain the tenant's full data, personal data included.
Store them where only operators can read them.

## Next

- [Tenants](../features/tenants.md): create, archive and delete tenants, and
  how each request picks one.
- [Migrate existing content](migrate-existing-content.md): bring content in
  from other systems, or move the whole database.
- [Object storage](../features/object-storage.md): the S3 bucket settings.
