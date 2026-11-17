---
title: "4. Write and publish"
description: "Write trips and their stops in LyEve Console or over the Content API, publish and unpublish them, keep earlier versions, restore one and discuss an entry with your team."
sidebar:
  order: 4
covers:
  console:
    - "/admin/content"
    - "/admin/content/[schema]"
    - "/admin/content/[schema]/new"
    - "/admin/content/[schema]/[id]"
    - "/admin/settings/content-history"
  api:
    - "GET /api/admin/content/history/settings"
    - "PUT /api/admin/content/history/settings"
    - "GET /api/admin/content/history/usage"
    - "POST /api/admin/content/history/prune"
    - "POST /api/v1/content/{schema}"
    - "GET /api/v1/content/{schema}"
    - "PUT /api/v1/content/{schema}/{id}/publish"
    - "PUT /api/v1/content/{schema}/{id}/unpublish"
    - "GET /api/v1/content/{schema}/{id}/revisions"
    - "PUT /api/v1/content/{schema}/{id}/revisions/{rev_id}/restore"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson you write the first trip of the travel guide and its stops, publish it, and
bring back an earlier version. You need the `trip` and `trip_stop` content types from
[lesson 3](model-content.md), with drafts turned on for `trip`, and a token in `TOKEN` for the
API steps.

## Keep earlier versions

> **Included free** on every install. Reading versions older than 30 days needs a license with
> the `content-pro` feature.

Turn history on before you write, so every save of a trip keeps the version it replaced.

<div class="paths">
<div data-path="console">

1. Open **Settings > Content history**.
2. Under **History**, set **Keep versions for** to **Selected types**, and pick **Trip** and
   **Trip stop** in **Content types that keep history**.
3. Keep **Keep each version for** at **Forever**, or choose **A number of days** and enter the
   **Days**. Set **Keep at most, per entry** the same way.
4. Under **Activity**, keep **Record who changed which entry** at **On**. Lesson 8 reads it.
5. Choose **Save**.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/content/history/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"history_mode": "selected", "schemas": ["trip", "trip_stop"], "keep_days": 90, "keep_versions": 20}'
```

`GET` on the same route reads the settings back.

</div>
</div>

![Content history settings with Selected types, Trip and Trip stop picked, the retention left at its defaults and Activity on.](./media/write-and-publish-history-settings.png)

**Storage** at the bottom says how much the stored versions take, which
`GET /api/admin/content/history/usage` also answers. **Prune now** runs the daily retention at
once (`POST /api/admin/content/history/prune`), and **Delete history** removes every stored
version of one content type. [Content history](../features/content-history.md) explains what
retention keeps.

## Find a content type

> **Included free** on every install.

Open **Content** in the sidebar. It lists every content type with its **Rows**, **Fields**,
**Relations** and **Last change**, and you sort by any of them. **Filter content types**
narrows the list by name, and a type with no entries yet can be hidden with **Hide them**. The
links at the top lead to **Releases**, **Activity** and **Schema builder**.

![The Content screen listing Trip and Trip stop with their row, field and relation counts.](./media/write-and-publish-content-list.png)

Choose **Trip** to see its entries. The **Status** filter shows **All**, **Draft**,
**Published** or **Archived** entries, and **Filter entries** searches the page.

## Write a trip

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. In **Content > Trip**, choose **New entry**.
2. Enter `Portugal by train` in **Title**, `portugal-by-train` in **Slug**, a few sentences in
   **Summary**, `2027-05-01` in **Starts on** and `Portugal` in **Country**. Leave **Cover**
   empty until lesson 5.
3. Choose **Create**. The trip appears in the list with the status `draft`.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3002/api/v1/content/trip \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "data": {
          "title": "Portugal by train",
          "slug": "portugal-by-train",
          "summary": "<p>Lisbon to Porto in a week, by rail.</p>",
          "starts_on": "2027-05-01",
          "country": "Portugal",
          "_status": "draft"
        }
      }'
```

The answer is `201` with the entry. An entry written through the Content API is published
unless its `data` names a `_status`. Keep its id for the next steps:

```bash
export TRIP_ID="<the id from the answer>"
```

</div>
</div>

![A new trip is written: the title, slug, summary, start date and country are filled in, and after Create the trip is listed as a draft.](./media/write-and-publish-new-trip.mp4)

Where locales are set up, **New entry** also asks for the **Locale** of the entry. Lesson 8 adds
a second one.

## Add the stops

> **Included free** on every install.

| Name | Day | Notes |
|---|---|---|
| `Lisbon` | 1 | Trams and the river |
| `Sintra` | 2 | A day trip by train |
| `Porto` | 4 | The last stop |

<div class="paths">
<div data-path="console">

1. Open **Content > Trip stop** and choose **New entry**.
2. Enter `Lisbon` in **Name**, `1` in **Day** and a note in **Notes**.
3. In **Trip**, type part of the title and pick **Portugal by train**.
4. Choose **Create**. Add `Sintra` and `Porto` the same way.

</div>
<div data-path="api">

A `belongs_to` field takes the other entry's id:

```bash
for stop in "Lisbon 1" "Sintra 2" "Porto 4"; do
  set -- $stop
  curl -X POST http://localhost:3002/api/v1/content/trip_stop \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"data": {"name": "'"$1"'", "day": '"$2"', "trip": "'"$TRIP_ID"'"}}'
done
```

Each answer is `201` and carries the link as `trip_id` under `data`.

</div>
</div>

![A stop is written: the name, day and notes are filled in, and typing in the Trip field offers Portugal by train, which is picked before Create.](./media/write-and-publish-stop-relation.mp4)

## Publish and unpublish

> **Included free** on every install.

A draft is invisible to readers of the Content API. Publishing it makes it readable.

<div class="paths">
<div data-path="console">

1. Open **Portugal by train** from **Content > Trip**. Its status reads `draft`.
2. Choose **Publish**. The status reads `published`.
3. To take it back, choose **Unpublish**. Publish it again before you go on.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3002/api/v1/content/trip/$TRIP_ID/publish \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `204`. Now a plain list returns the trip:

```bash
curl http://localhost:3002/api/v1/content/trip \
  -H "Authorization: Bearer $TOKEN"
```

`PUT /api/v1/content/trip/$TRIP_ID/unpublish` takes it back to draft.

</div>
</div>

![Publish is chosen on a draft trip, the status changes to published, and the trip shows as published in the list.](./media/write-and-publish-publish.mp4)

[Content lifecycle](../features/content-lifecycle.md) covers archiving, scheduled publishing
and publishing many entries at once.

## Restore an earlier version

> **Included free** on every install. Reading versions older than 30 days needs a license with
> the `content-pro` feature.

<div class="paths">
<div data-path="console">

1. Open the trip, change the **Title** to `Portugal by rail` and choose **Save**.
2. Open it again and choose **History**. The count beside it says how many versions are kept.
3. Under **Revision history**, pick the version from before the change. It shows when it was
   saved and what changed.
4. Choose **Restore**. The title reads `Portugal by train` again, and the restore is itself a
   new version.

</div>
<div data-path="api">

```bash
curl http://localhost:3002/api/v1/content/trip/$TRIP_ID/revisions \
  -H "Authorization: Bearer $TOKEN"
```

Pick the `id` of the revision you want, then restore it:

```bash
curl -X PUT http://localhost:3002/api/v1/content/trip/$TRIP_ID/revisions/<rev_id>/restore \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![The trip's history panel opens, an earlier revision is picked, and Restore brings back the old title.](./media/write-and-publish-restore.mp4)

A content type with history off says **History is off for this type.** in place of the panel,
with a link to turn it on.

## Discuss an entry

> **Requires** a license with the `content-pro` feature.

Comment threads sit beside the entry, so a question about a trip stays with the trip. This is
a console task here, and [Editorial comments](../features/editorial-comments.md) shows the
routes behind it.

1. Open the trip. In the comments card, write under **New thread** and choose **Comment**.
2. Pick people in **Mention** to email them about the thread.
3. Answer under **Reply**. Resolve the thread once it is settled.

On a free install, the entry has no comments card. With the license, the card sits beside the
fields.

![A trip on a free install: the fields, status, history and translations, with no comments card.](./media/write-and-publish-comments-free.png)

![A trip with the license: a new thread is written, a colleague is mentioned, and a reply follows.](./media/write-and-publish-comments-licensed.mp4)

The same editor holds three more cards that later lessons use: **Translations** in lesson 8,
the review bar in lesson 6, and **Add to release** in lesson 6.

## Next

- [5. Media](media.md): upload a cover photo for the trip.
- [Content lifecycle](../features/content-lifecycle.md): statuses, schedules, relations and
  bulk publishing.
- [Content history](../features/content-history.md): retention, restore and the activity log.
- [Editors](../guides/user/index.md): the console for people who write content every day.
