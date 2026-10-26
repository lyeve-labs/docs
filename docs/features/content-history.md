---
title: Content history
description: Choose which content types keep earlier versions of their entries, how long they keep them, and read who changed what in the content activity log.
sidebar:
  order: 15
capability_of: content
---

> **Included free** on every install.

Content history keeps earlier versions of an entry, so you can compare two and bring an old
one back. It is off by default. You turn it on for every content type, for the types you
choose, or for one, and you decide how long versions are kept. Beside it, the content
activity log records who changed which entry, when, from where and which fields.

:::tip[Best for]
- **Undoing a bad edit.** Compare the current entry with the version before it and restore
  the good one.
- **Keeping history only where it matters.** Keep every version of your legal pages and none
  of a high-traffic type your app rewrites every minute.
- **Bounding what history stores.** Keep 90 days, or the last 20 versions of each entry, and
  let the daily run remove the rest.
- **Answering "who changed this?"** Read every change to an entry, a content type or a
  person's work, across every tenant when you are a super admin.
:::

:::note[Use something else when]
- You need a tamper-evident record of every action on the instance. The
  [audit log](audit-log.md) chains its entries and exports them.
- You want to publish many entries at once and take them back together. Use
  [content releases](content-releases.md).
:::

## In the admin console

Open **Settings > Content history**. You need the `admin` or `super_admin` role.

- Under **History**, **Keep versions for** is **Off**, **All types** or **Selected types**.
  With **Selected types**, tick the types in **Content types that keep history**.
- **Keep each version for** is **Forever** or a number of days, and **Keep at most, per
  entry** is **Unlimited** or a number of versions.
- Under **Activity**, **Record who changed which entry** turns the activity log on or off,
  with its own **Keep activity for**.
- **Storage** shows how much the stored versions take, beside what the same versions would
  take as full copies, with **Prune now** and **Delete history** for one content type.

When a setting is fixed by the server's environment, a banner says so and its control
cannot be changed. On an entry whose type keeps no history, the entry page says
**History is off for this type** and links an administrator to these settings.

Read the activity log under **Insight > Content activity**.

## How it works

A version is recorded when an entry of a type that keeps history changes: on a save in the
console, a write on the Admin API or the Content API, GraphQL or gRPC, a bulk import, a flow,
a release or a scheduled publish. The numbered revisions, comparing and restoring work as
described in [Content lifecycle](content-lifecycle.md#compare-and-roll-back-revisions).

History stays small:

- **A save that changes nothing stores nothing.** It records no version and does not move
  the entry's revision number.
- **An edit stores only the fields that changed.** Every field you did not touch is shared
  with the version before it, so changing a title on a long article stores the title.
- **A change of status alone stores no field values.** Publishing in the console, in a
  release or on a schedule records a version that points at values already stored. A publish
  or unpublish made by another writer, such as the Content API, GraphQL, gRPC, a flow or an
  approval, records no version. The activity log records who made it.
- **Restoring an old version stores no new values.** The restored version points at the
  values the old one held.
- **Values are compressed** when that makes them smaller.

An entry's `current_rev` moves on every real change, whether history is on or not, so
version numbers can skip. They skip where history was off for the type, where only the slug
or the schedule changed, and where retention removed older versions.

Turning history off stops recording new versions and keeps the ones already stored. They
stay readable and restorable until retention or **Delete history** removes them. Deleting an
entry deletes its versions with it.

## Choose the content types

| Mode | What keeps history |
|---|---|
| `off` | No content type. |
| `all` | Every content type, including ones created later. |
| `selected` | The content types named in `schemas`. |

To keep history for one type, choose `selected` and name only that type. A renamed content
type stays selected under its new name, and a deleted one leaves the list.

A new install and a new tenant start with history off. When you upgrade, a tenant that
already holds content keeps history on for every content type, so nothing it did before the
upgrade changes. Its retention values stay unset, which means the environment defaults
below, and forever when those are unset too.

## Settings

Five environment variables set history for the whole instance. Set them on the engine.

| Variable | Values | Default | Effect |
|---|---|---|---|
| `CONTENT_HISTORY` | `console`, `off`, `on` | `console` | `console` lets each tenant choose in the console. `off` turns history off for every tenant. `on` turns it on for every content type of every tenant. |
| `CONTENT_HISTORY_KEEP_DAYS` | A number of days, `0` for forever | `0` | How long a version is kept, for a tenant that has not chosen. |
| `CONTENT_HISTORY_KEEP_VERSIONS` | A number of versions, `0` for all | `0` | How many versions of each entry are kept, for a tenant that has not chosen. |
| `CONTENT_ACTIVITY` | `console`, `off`, `on` | `console` | The same three values, for the activity log. |
| `CONTENT_ACTIVITY_KEEP_DAYS` | A number of days, `0` for forever | `0` | How long an activity row is kept, for a tenant that has not chosen. |

`off` and `on` lock the matching control in the console. With `CONTENT_HISTORY=on`, a
tenant cannot turn history off or narrow it to some types, but it still sets its own
retention. A `PUT` that asks for a locked value other than the one in effect answers `409`.
The three `KEEP` variables are defaults and lock nothing: a tenant's own choice wins over
them. A value the engine cannot read, such as `yes` or `-1`, is ignored with a warning in
the log, and the default applies.

## Try it

This turns history on for one content type, edits an entry and reads what was kept.
The settings routes need a signed-in session with the `admin` or `super_admin` role, so put
the token from your login answer in `TOKEN`, as the [quickstart](../getting-started/quickstart.md)
shows. An [admin token](../how-to/admin-tokens.md) cannot change these settings.

1. Read the settings:

   ```bash
   curl http://localhost:3001/api/admin/content/history/settings \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   {
     "settings": {
       "history_mode": "off",
       "schemas": [],
       "keep_days": 0,
       "keep_versions": 0,
       "activity_enabled": true,
       "activity_keep_days": 0,
       "updated_by": null,
       "updated_at": null
     },
     "locked": [],
     "env": { "history": "console", "activity": "console" },
     "defaults": { "keep_days": 0, "keep_versions": 0, "activity_keep_days": 0 }
   }
   ```

   `locked` names the fields the environment fixes, `env` holds `CONTENT_HISTORY` and
   `CONTENT_ACTIVITY`, and `defaults` holds the three `KEEP` variables.

2. Keep history for the `story` type only, for 90 days and at most 20 versions an entry:

   ```bash
   curl -X PUT http://localhost:3001/api/admin/content/history/settings \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"history_mode": "selected", "schemas": ["story"], "keep_days": 90, "keep_versions": 20}'
   ```

   The answer has the same shape as the read. A field you leave out keeps its value.

3. Edit a `story` entry, such as the one [Content lifecycle](content-lifecycle.md#try-it)
   creates, with its id in `ENTRY_ID`. Then list its versions:

   ```bash
   curl -X PUT http://localhost:3001/api/admin/content/$ENTRY_ID \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"title": "Hello again"}'

   curl http://localhost:3001/api/admin/content/$ENTRY_ID/revisions \
     -H "Authorization: Bearer $TOKEN"
   ```

   The list carries `"history_enabled": true`, and each version names its `action`, such as
   `updated`, and its `source`, such as `console` or `api`. Sending the same title again
   adds no version.

4. See what history takes:

   ```bash
   curl http://localhost:3001/api/admin/content/history/usage \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   {"commits": 1, "objects": 2, "stored_bytes": 412, "logical_bytes": 1730, "legacy_rows": 0, "legacy_bytes": 0, "oldest_at": "2026-10-22T09:14:03Z"}
   ```

   `commits` counts versions, `stored_bytes` is what they take, and `logical_bytes` is what
   the same versions would take as full copies. `legacy_rows` and `legacy_bytes` count the
   revisions recorded before your instance stored history this way. Retention removes those
   too. `oldest_at` is the time of the oldest version.

## Retention

Retention decides how long versions are kept. Each tenant sets it in the console or with
`PUT /api/admin/content/history/settings`. A value of `0` means no limit.

| You want | Set |
|---|---|
| Keep every version forever | `keep_days` `0` and `keep_versions` `0` |
| Keep 90 days | `keep_days` `90` |
| Keep the last 20 versions of each entry | `keep_versions` `20` |
| Both | `keep_days` `90` and `keep_versions` `20`. A version is removed once it breaks either limit |

`activity_keep_days` does the same for the activity log. Retention past 30 days is allowed
on every install. Reading versions older than 30 days needs `content-pro`, as
[below](#how-far-back-you-can-read).

Retention never removes:

- The newest version of an entry, whatever its age.
- Any version of an entry that an unfinished release or a pending scheduled change names.
- A version written by a person an active [legal hold](audit-log.md#retention-policies-and-legal-holds)
  names, whichever tenant the hold was set in.
- Any version of an entry an active legal hold names.

A legal hold that names no actor and no single resource id covers the tenant's content as a
whole when its resource type is empty or `content_entry`. Such a hold can still be limited to
an action or a time window. While one is active, retention deletes nothing in the tenant,
neither versions nor activity rows, and the daily run skips the tenant.

Retention runs on its own a minute after the engine starts and then every 24 hours, on one
engine at a time when you run several. It also covers a tenant whose history is now off,
with that tenant's last retention values, and the revisions recorded before your instance
stored history this way.

### Prune now

**Prune now** in the console, or `POST /api/admin/content/history/prune` with `{}`, runs
retention for your tenant at once. It answers what it removed:

```json
{"commits": 140, "legacy_rows": 12, "activity_rows": 0, "done": true}
```

`commits` counts the versions removed, `legacy_rows` the revisions recorded before your
instance stored history this way, and `activity_rows` the activity rows removed.

`200` with `"done": true` means the run finished. A run that takes longer than about ten
seconds answers `202` with `"done": false` and the counts so far, and keeps going in the
background. Read the usage again later to see the result. A second prune while one runs
answers `409`. So does a prune while a legal hold covers the tenant's content as a whole, and
it deletes nothing.

### Delete one content type's history

**Delete history** in the console, or
`DELETE /api/admin/content/history?schema=story`, deletes every stored version of one content
type. The entries stay as they are. The deleted versions cannot be recovered.

```json
{"commits": 312, "held": 4}
```

`held` counts the versions a legal hold kept. They stay until the hold is lifted. The delete
is recorded in the activity log. While a legal hold covers the tenant's content as a whole, the
delete removes nothing and answers `409`.

## Compare and restore versions

Open an entry and choose **History** to list its versions, then **Restore** to bring one
back. On the Admin API, `GET /api/admin/content/{id}/revisions` lists them,
`GET /api/admin/content/{id}/diff?from=1&to=3` compares two, and
`POST /api/admin/content/{id}/rollback` restores one. A restore takes the old version's
title, body, meta and status, and is recorded as a new version when history is on.
[Content lifecycle](content-lifecycle.md#compare-and-roll-back-revisions) covers each route.

The Content API keeps its own versions of an entry, each taken before an update when history
is on for the type. `GET /api/v1/content/{schema}/{id}/revisions` lists them and
`PUT /api/v1/content/{schema}/{id}/revisions/{rev_id}/restore` writes one back. The list is
empty when the type has no stored versions.

### How far back you can read

Without `content-pro`, you list, read, compare and restore the versions of the last 30 days,
plus the newest version of each entry whatever its age. Older versions are kept for as long
as your retention says, and reading them answers `402` naming `feature:content-pro`. With
`content-pro` every kept version reads at once. A license that lapses deletes nothing.

## Activity log

The activity log records each change to an entry: who made it, when, the content type and
entry, the action, the source and the names of the fields that changed. A row names the
version history recorded for that change, when there is one. Activity runs on its own,
apart from history, so it can stay on while history is off. It is on by default.

| Field | Values |
|---|---|
| `action` | `created`, `updated`, `published`, `unpublished`, `deleted`, `rollback`, `scheduled_publish`, `scheduled_unpublish`, and `history_settings` and `history_deleted` for the settings |
| `source` | `console`, `api` for the Admin API, the Content API, GraphQL, gRPC and bulk writes over the API, `flow`, `release`, `schedule`, `import`, and `backfill` for an entry the content screens picked up after another writer created it |
| `actor_id`, `actor_email` | Who made the change, or `null` when no one is known |
| `actor_kind` | How the writer signed in: `user` for a session, `admin_token` or `api_key`. `null` when the writer is unknown |
| `fields` | The names of the fields that changed |
| `revision_num` | The version history recorded, or `null` |

Who a row names:

- A change from the console or with a session names that person.
- A change with an [admin token](../how-to/admin-tokens.md) names the token's owner.
- A change with an [API key](api-keys.md) names the key's id, and `actor_email` is `null`
  because no account matches it.
- A change a flow makes names no person yet. Its `source` is `flow`.

`deleted` is a delete of the entry. Archiving an entry records `updated`, with `status` in
`fields`. A delete recorded before the upgrade is stored as `archived`, and the activity
routes answer it as `deleted`, so `action=deleted` finds both. Rows recorded before the
upgrade have a `null` `actor_kind` and keep their `actor_id`.

A change to the history settings and a history delete are always recorded, even with
activity off. Turning activity off stops every other row.

Read the log with `GET /api/admin/content/activity`. It takes `schema`, `entry_id`,
`actor`, `action`, `source`, `from` (inclusive) and `to` (exclusive), as RFC 3339 times,
with `limit` and `offset`, and answers `{"data": [...], "total_count": n, "limit": n, "offset": n}`,
newest first.
An `admin` or `super_admin` reads it, and so does an admin token with the `content:read`
grant.

```bash
curl "http://localhost:3001/api/admin/content/activity?schema=story&action=updated&limit=20" \
  -H "Authorization: Bearer $TOKEN"
```

A super admin reads every tenant's activity with `GET /api/admin/content/activity/all`,
which takes the same filters plus `tenant_id`. Leave `tenant_id` empty for every tenant. In
the console, **Content activity** shows a super admin a tenant picker with **All tenants**.

With [PII masking](pii-masking.md) on, an admin reads `actor_email` masked. A super admin
reads it in full on both routes, and the masking access log records each of those reads.

The audit log, when your license carries `audit`, keeps receiving its own record of each
create, update and delete, and of each change to these settings.

## Upgrade and downgrade

Upgrading adds history's storage beside what your database holds and rewrites nothing, so
it runs quickly on a large database. The revisions your instance kept before stay readable
and restorable, list in order with the new versions, and are pruned by the same retention.

There is no downgrade to an engine from before content history. An earlier engine refuses to
start on a database this release has upgraded, rather than run on data it cannot delete. The
way back is the backup you took before the upgrade. Back up your database before you
upgrade, as for every release.

## Routes

| Method | Path | Does | Who |
|---|---|---|---|
| `GET` | `/api/admin/content/history/settings` | The tenant's settings, what the environment locks, and the environment defaults | `admin`, signed-in session |
| `PUT` | `/api/admin/content/history/settings` | Change them. A field you leave out keeps its value | `admin`, signed-in session |
| `GET` | `/api/admin/content/history/usage` | What the tenant's history stores | `admin`, or `content:read` |
| `POST` | `/api/admin/content/history/prune` | Run retention for the tenant now | `admin`, signed-in session |
| `DELETE` | `/api/admin/content/history?schema=<name>` | Delete every version of one content type | `admin`, signed-in session |
| `GET` | `/api/admin/content/activity` | The tenant's activity | `admin`, or `content:read` |
| `GET` | `/api/admin/content/activity/all` | Every tenant's activity, or one tenant's with `tenant_id` | `super_admin` |

`admin` here also means `super_admin`.

## Errors

| Status | Message |
|---|---|
| `400` | `invalid JSON body` |
| `400` | `entry_id must be a UUID`, `actor must be a UUID`, `from must be an RFC 3339 time`, `to must be an RFC 3339 time` |
| `402` | `payment_required`, naming `feature:content-pro`, for a version older than 30 days without `content-pro` |
| `403` | `the admin role is required` on the settings, prune and delete routes |
| `403` | `super_admin required` on `/api/admin/content/activity/all` |
| `409` | `history_mode is locked by the server configuration` or `activity_enabled is locked by the server configuration` |
| `409` | `a history prune is already running`, while another prune runs for the tenant |
| `409` | `a legal hold covers this tenant's content, so history was not pruned`, or `so history was not deleted` on the delete |
| `422` | `history_mode must be off, all or selected` |
| `422` | `schemas holds too many names`, `each schema name is between 1 and 255 characters` |
| `422` | `keep_days must be between 0 and 36500`, `keep_versions must be between 0 and 1000000`, `activity_keep_days must be between 0 and 36500` |
| `422` | `schema is required and at most 255 characters` on the delete |
| `503` | `failed to prune history` when the database fails during a prune |
| `503` | `legal hold status is unavailable, so history was not pruned`, or `so history was not deleted` on the delete |

## Related

- [Content lifecycle](content-lifecycle.md): statuses, revisions, comparing and rolling back
  over the Admin API.
- [Drafts and publishing](../guides/user/drafts-and-publishing.md): the history panel in
  the console.
- [Audit log](audit-log.md): the instance-wide record, and legal holds.
