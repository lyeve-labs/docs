---
title: Content lifecycle
description: Move entries through draft, published and archived over the Admin API, compare and roll back revisions, schedule publishing, and link entries.
sidebar:
  order: 16
---

> **Included free** on every install, with every revision kept and the last 30 days of them readable. Older revisions, content releases and editorial comment threads need a license with the `content-pro` feature. See [pricing](https://lyeve.com/pricing/).

This is the Admin API behind the admin console's **Content** screens. It creates and edits
entries, moves them between draft, published and archived, and keeps a numbered revision of
every write so you can compare two and roll back. An entry can also publish and archive
itself at times you set, and link to other entries.

:::tip[Best for]
- **Publishing from a script.** Create drafts from a CI job or an import, and publish them
  when a person or a check says they are ready.
- **Undoing a bad edit.** See what changed between two revisions and roll back to the good
  one. The rollback is itself a revision, so nothing is lost.
- **Timed launches and embargoes.** Publish a page at 09:00 Paris time and archive it a
  month later, with no one at a keyboard.
- **Answering "who changed this?"** Every create, edit, publish, rollback and schedule is
  recorded with the account that made it.
- **Linking entries outside the content model.** Point an entry at related ones and walk the
  links as a graph.
:::

:::note[Use something else when]
- Changes need sign-off before they go live. [Editorial review](editorial-review.md) adds
  stages, a role per stage and publishing on final approval.
- Your site or app writes content. It uses the Content API, described on
  [the data model](../concepts/data-model.md#entries) page.
:::

## In the admin console

Open **Content**, pick a content type and open an entry. The badge at the top shows its
status, beside **Publish** or **Unpublish**. **History** opens the **Revision history** panel,
and the number beside it counts the revisions. Choose a revision, then **Restore** to bring it
back as a new revision.

![An entry's page with the Revision history panel open beside it. The entry is published, with Unpublish and History (3) at the top, and the oldest of its three revisions is selected with a Restore button under it.](images/content-lifecycle.png)

[Drafts and publishing](../guides/user/drafts-and-publishing.md) walks through these steps.
Archiving, scheduling, comparing two revisions and change notes are on the Admin API, below.

## How it works

An entry on this API has a few fields of its own beside the content type's field values:

| Field | What it holds |
|---|---|
| `schema` | The content type the entry belongs to. |
| `title`, `slug` | Required. A slug is unique across your tenant, not only within one content type. |
| `body` | The content type's field values. They are checked against its rules on every write. |
| `meta` | Any JSON you want to keep beside the entry. |
| `status` | `draft`, `published` or `archived`. A new entry is a `draft` unless you say otherwise. |
| `current_rev` | The number of the newest revision. |
| `published_at` | When the entry last moved to published. |
| `scheduled_publish_at`, `scheduled_unpublish_at`, `timezone` | Its schedule, when one is set. |

:::caution[Drafts reach the Content API as published]
Today the Content API serves an entry written here as `published`, whatever its `status`. The
status holds in the console and on this API only. See
[the data model](../concepts/data-model.md#statuses).
:::

What statuses and revisions mean is explained on [the data model](../concepts/data-model.md)
page. Every route takes the `admin` or `super_admin` role. An
[admin token](../how-to/admin-tokens.md) can call the read routes with the `content:read`
grant and the write routes with `content:write`.

## Try it

This creates a draft, publishes it, rolls it back and schedules it. You need a token in
`TOKEN`. The [quickstart](../getting-started/quickstart.md) shows how to get one.

1. Create a `post` content type with draft and publish:

   ```bash
   curl -X POST http://localhost:3001/api/admin/schemas \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name": "post", "with_draft_publish": true, "fields": [{"name": "title", "field_type": "text", "required": true}, {"name": "body", "field_type": "rich_text"}]}'
   ```

2. Create an entry. The content type requires `title`, so it goes in `body` as well as beside
   it:

   ```bash
   curl -X POST http://localhost:3001/api/admin/content \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"schema": "post", "slug": "hello", "title": "Hello", "body": {"title": "Hello", "body": "<p>First post.</p>"}}'
   ```

   ```json
   {
     "id": "ad39fb3d-eb5a-47ce-9cd5-e7d421acaa98",
     "schema": "post",
     "tenant_id": "default",
     "slug": "hello",
     "title": "Hello",
     "body": { "title": "Hello", "body": "<p>First post.</p>" },
     "meta": {},
     "status": "draft",
     "timezone": "UTC",
     "created_by": "12dd433d-c110-4809-b476-12ef360afe7f",
     "updated_by": "12dd433d-c110-4809-b476-12ef360afe7f",
     "current_rev": 1,
     "created_at": "2026-10-02T05:06:17.627726Z",
     "updated_at": "2026-10-02T05:06:17.627726Z"
   }
   ```

   Copy its `id` into `ENTRY_ID`.

3. Publish it, with a note for the history:

   ```bash
   curl -X PUT http://localhost:3001/api/admin/content/$ENTRY_ID \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"status": "published", "change_note": "First release"}'
   ```

   The answer is `200` with `"status": "published"`, a `published_at` time and
   `"current_rev": 2`.

4. List the revisions, newest first:

   ```bash
   curl http://localhost:3001/api/admin/content/$ENTRY_ID/revisions \
     -H "Authorization: Bearer $TOKEN"
   ```

   You see revision `2` with the note `First release` and revision `1` with the note
   `initial revision`.

5. Roll back to revision 1:

   ```bash
   curl -X POST http://localhost:3001/api/admin/content/$ENTRY_ID/rollback \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"to_revision": 1}'
   ```

   ```json
   {
     "message": "rolled back to revision 1",
     "new_revision": {
       "id": "fc1659a6-afa4-48e5-9089-b786fbb45d94",
       "entry_id": "ad39fb3d-eb5a-47ce-9cd5-e7d421acaa98",
       "tenant_id": "default",
       "revision_num": 3,
       "title": "Hello",
       "body": { "title": "Hello", "body": "<p>First post.</p>" },
       "meta": {},
       "status": "draft",
       "change_note": "rolled back to revision 1",
       "created_by": "12dd433d-c110-4809-b476-12ef360afe7f",
       "created_at": "2026-10-02T05:06:34.085365Z"
     }
   }
   ```

6. Schedule it to publish and then archive:

   ```bash
   curl -X PUT http://localhost:3001/api/admin/content/$ENTRY_ID/entry-schedule \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"scheduled_publish_at": "2027-11-01T09:00:00+01:00", "scheduled_unpublish_at": "2027-12-01T09:00:00+01:00", "timezone": "Europe/Paris"}'
   ```

   The answer is the entry with `scheduled_publish_at` and `scheduled_unpublish_at` stored in
   UTC.

## Find and read entries

`GET /api/admin/content` lists entries, newest change first, 50 at a time by default and at
most 500. Narrow it with `?schema=post` and `?status=published`, and page with `limit` and
`offset`. This route filters by content type and status only: `filters[...]` answers `400`
and points you to the Content API, which filters by field.

Read one entry by id with `GET /api/admin/content/{id}`, or by slug with
`GET /api/admin/content/slug/{slug}`. With [localization](localization.md), each read takes
`?locale=fr` and answers the translated fields.

## Edit, publish and archive

`PUT /api/admin/content/{id}` changes only the top-level fields you send, so
`{"status": "published"}` leaves the title and body as they are. A `body` you send replaces
the whole body, so send every field value, not only the one that changed. Every write saves a
new revision. Send `change_note` to describe it. The note is `updated` when you leave it out.

Publishing, unpublishing and archiving are changes of `status` on the same route:

| To | Send |
|---|---|
| Publish | `{"status": "published"}` |
| Take back to draft | `{"status": "draft"}` |
| Archive | `{"status": "archived"}` |

You can also create an entry as published by sending `"status": "published"` on the create.
`DELETE /api/admin/content/{id}` removes the entry and all its revisions, and answers `204`.

## Compare and roll back revisions

| Method | Path | Result |
|---|---|---|
| `GET` | `/api/admin/content/{id}/revisions` | The revisions you can read, newest first, with `limit` and `offset`. |
| `GET` | `/api/admin/content/{id}/revisions/{number}` | One revision: its title, body, meta, status and change note. |
| `GET` | `/api/admin/content/{id}/diff?from=1&to=3` | The fields that changed between two revisions, with their old and new values, as text in `diff`. |
| `POST` | `/api/admin/content/{id}/rollback` | `{"to_revision": 2}`. The entry takes that revision's title, body, meta and status, saved as a new revision. |

### How far back you can read

Every revision of every entry is kept. Without `content-pro`, you list, read,
compare and roll back to the revisions of the last 30 days, plus the newest
revision of each entry whatever its age, so a page nobody has touched in a year
still shows its current state. The list says what it left out:

```json
{"data": [...], "total_count": 4, "limit": 50, "offset": 0, "hidden_older": 12, "window_days": 30}
```

`hidden_older` counts the older revisions, and `total_count` counts only the
ones you can page through. Reading, comparing or rolling back to an older
revision answers `402` naming `feature:content-pro` and writes nothing. The
Content API's revision list and restore follow the same window.

With `content-pro`, `hidden_older` is `0`, `window_days` is `null`, and every
revision reads at once, however old. A license that lapses deletes nothing,
and the older history comes back with it.

## Schedule a publish or an archive

`PUT /api/admin/content/{id}/entry-schedule` sets `scheduled_publish_at`,
`scheduled_unpublish_at`, or both, with an optional `timezone`:

- Times are RFC 3339 and must be in the future. `timezone` is an IANA name such as
  `Europe/Paris`, kept with the schedule for display.
- At `scheduled_publish_at` a draft becomes published. At `scheduled_unpublish_at` a
  published entry becomes archived.
- Due changes are applied every 30 seconds, so an entry can change up to 30 seconds after its
  time.

`DELETE` on the same path clears the schedule.
`GET /api/admin/content/{id}/preview?at=2027-11-02T00:00:00Z` answers the status the entry
will have at that moment, as `effective_status`, with `will_publish` and `will_unpublish`.
`POST /api/admin/content/{id}/schedule-conflicts` checks proposed times against other
entries and answers `{"has_conflicts": true, "conflicts": [...]}`.
`GET /api/admin/content/scheduled` lists every entry with a schedule.

To queue one change on its own, `POST /api/admin/content/{id}/schedule` takes
`{"target_status": "published", "publish_at": "2027-11-01T08:00:00Z"}`, where `target_status`
is `published` or `archived`. `GET /api/admin/content/{id}/schedules` lists them, and
`DELETE /api/admin/content/{id}/schedules/{schedule_id}` cancels one.

## Link entries to each other

A relationship links two entries directly, without a relation field in the content type. Use
it for links an editor decides case by case, such as "see also". For links every entry of a
type has, such as a post's author, use a [relation field](../concepts/data-model.md#relations).

```bash
curl -X POST http://localhost:3001/api/admin/content/$ENTRY_ID/relationships \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"target_id": "<another entry id>", "field_name": "see_also", "rel_type": "many_to_many", "cascade_policy": "none"}'
```

| Field | Values |
|---|---|
| `target_id` | The linked entry. Required. |
| `field_name` | The name of the link. Required. |
| `rel_type` | `one_to_one`, `one_to_many` or `many_to_many`. |
| `bidirectional` | `true` to make the link visible from both ends. |
| `cascade_policy` | `none`, or `cascade` to delete the target when the source is deleted. |
| `sort_order` | Position among links with the same name. |

`GET /api/admin/content/{id}/relationships` lists the links from an entry, filtered with
`?field=`. `.../relationships/reverse` lists the links to it, and `.../graph?depth=3` answers
the linked entries as `nodes` and `edges`, up to `depth` steps.

## See who changed an entry

`GET /api/admin/content/{id}/audit` lists every change to the entry, newest first. Each item
names the `action` (`created`, `updated`, `published`, `unpublished`, `archived`, `rollback`,
`scheduled_publish` or `scheduled_unpublish`), the `actor_id`, the `revision_num` and, for
edits, a `diff`. The instance-wide record is the [audit log](audit-log.md).

## Publish many entries at once

With a license that carries `content-pro`, a release groups up to 500 entries
that publish or unpublish together, now or at a set time, and puts every entry
back if any fails. A release scheduled while the license covered it still goes
out after a lapse. See [Content releases](content-releases.md).

## Discuss an entry

With `content-pro`, editors can also keep comment threads on an entry, reply,
resolve and reopen them, and mention colleagues by email. Reading, resolving
and reopening stay free after a lapse. See
[Editorial comments](editorial-comments.md).

## Settings

Content lifecycle has no settings of its own. It runs on every install. If you set
`LYEVE_PLUGINS` to choose which features start, include `content` in it. See
[licensing and tiers](../concepts/licensing-and-tiers.md).

## Routes

<details>
<summary>Entries, revisions, schedules, relationships and history</summary>

| Method | Path | Purpose | Grant |
|---|---|---|---|
| `GET` | `/api/admin/content` | List entries. `?schema=`, `?status=`, `limit`, `offset`. | `content:read` |
| `POST` | `/api/admin/content` | Create an entry. | `content:write` |
| `GET` | `/api/admin/content/{id}` | One entry. | `content:read` |
| `GET` | `/api/admin/content/slug/{slug}` | One entry by slug. | `content:read` |
| `PUT` | `/api/admin/content/{id}` | Change an entry and save a revision. | `content:write` |
| `DELETE` | `/api/admin/content/{id}` | Delete an entry and its revisions. | `content:write` |
| `GET` | `/api/admin/content/{id}/revisions` | Revisions, newest first. | `content:read` |
| `GET` | `/api/admin/content/{id}/revisions/{number}` | One revision. | `content:read` |
| `GET` | `/api/admin/content/{id}/diff` | Compare two revisions: `from`, `to`. | `content:read` |
| `POST` | `/api/admin/content/{id}/rollback` | Restore a revision: `to_revision`. | `content:write` |
| `PUT`, `DELETE` | `/api/admin/content/{id}/entry-schedule` | Set or clear the publish and archive times. | `content:write` |
| `GET` | `/api/admin/content/{id}/preview` | The status at a moment: `at`. | `content:read` |
| `POST` | `/api/admin/content/{id}/schedule-conflicts` | Check proposed times against other entries. | `content:read` |
| `GET` | `/api/admin/content/scheduled` | Entries with a schedule. | `content:read` |
| `POST` | `/api/admin/content/{id}/schedule` | Queue one change of status. | `content:write` |
| `GET` | `/api/admin/content/{id}/schedules` | Queued changes. | `content:read` |
| `DELETE` | `/api/admin/content/{id}/schedules/{schedule_id}` | Cancel a queued change. | `content:write` |
| `GET` | `/api/admin/content/{id}/audit` | Every change to the entry. | `content:read` |
| `GET`, `POST` | `/api/admin/content/{id}/relationships` | List or add links from the entry. | read or write |
| `GET` | `/api/admin/content/{id}/relationships/reverse` | Links to the entry. | `content:read` |
| `GET`, `PUT`, `DELETE` | `/api/admin/content/{id}/relationships/{rel_id}` | Read, change or remove one link. `PUT` changes `cascade_policy` and `sort_order`. | read or write |
| `GET` | `/api/admin/content/{id}/graph` | Linked entries as a graph: `depth`. | `content:read` |

</details>

## Errors

The ones you meet most:

- `400` `slug is required` or `title is required` on a create.
- `409` when the slug is taken. The message names the content type that holds it, such as
  `slug is already taken by content type "page"; a slug is unique per tenant, not per content type`.
- `422` with an `errors` list when `body` breaks the content type's rules, such as a required
  field left out.

<details>
<summary>Every error these routes return</summary>

| Status | Message |
|---|---|
| `400` | `invalid JSON body` |
| `400` | `slug is required`, `title is required`, `slug must not contain control characters`, `slug must not contain a percent sign` |
| `400` | `field filters are not supported on this route; ...` |
| `400` | `from and to query params required (positive integers)` |
| `400` | `invalid revision number`, `to_revision must be positive` |
| `402` | `payment_required`, naming `feature:content-pro`, for a revision older than 30 days without `content-pro` |
| `400` | `target_status is required`, `publish_at is required`, `publish_at must be in the future`, `target_status must be published or archived` |
| `400` | `at least one of scheduled_publish_at or scheduled_unpublish_at is required` |
| `400` | `scheduled_publish_at must be in the future`, `scheduled_unpublish_at must be in the future` |
| `400` | `invalid timezone: <name>` |
| `400` | `` `at` query parameter is required (ISO 8601 timestamp) `` |
| `400` | `target_id is required`, `field_name is required`, `invalid target_id` |
| `404` | `content not found`, `revision not found`, `relationship not found`, `schedule not found or already fired` |
| `409` | `slug is already taken by another entry of this content type`, or by another content type as above |
| `422` | An `errors` list naming each field that breaks the content type's rules |
| `422` | `scheduled_publish_at or scheduled_unpublish_at is required` on schedule-conflicts |

The release and comment errors are on [Content releases](content-releases.md#errors) and
[Editorial comments](editorial-comments.md#errors).

</details>

## Related

- [Drafts and publishing](../guides/user/drafts-and-publishing.md): the same lifecycle in the
  admin console.
- [Data model](../concepts/data-model.md): statuses, revisions and the Content API.
- [Editorial review](editorial-review.md): stages and approval before publishing.
- [Bulk import](bulk-import.md): create many entries from a file.
