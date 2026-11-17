---
title: "6. Reviews and releases"
description: "Define an editorial review for trips, send a trip through it and approve it, then group entries into a release and publish them together, now or at a set time."
sidebar:
  order: 6
covers:
  console:
    - "/admin/reviews"
    - "/admin/reviews/definitions"
    - "/admin/releases"
    - "/admin/releases/[id]"
  api:
    - "POST /api/admin/review/definitions"
    - "DELETE /api/admin/review/definitions/{id}"
    - "GET /api/admin/review/assignments"
    - "POST /api/admin/review/assignments"
    - "POST /api/admin/review/assignments/{id}/transition"
    - "GET /api/admin/content/releases"
    - "POST /api/admin/content/releases"
    - "POST /api/admin/content/releases/{id}/items"
    - "POST /api/admin/content/releases/{id}/schedule"
    - "POST /api/admin/content/releases/{id}/publish"
    - "GET /api/admin/content/releases/{id}/conflicts"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson a trip waits for a second pair of eyes before it goes live, and then several
entries go live at once. You need the trip from [lesson 4](write-and-publish.md) in `TRIP_ID`,
on a content type with drafts turned on as [lesson 3](model-content.md) did, and a token in
`TOKEN` for the API steps.

## Define a review

> **Included free** on every install, with one review definition per tenant. More definitions,
> and approvals from several reviewers, escalation and conditions on a stage, need a license
> with the `review-pro` feature.

A review definition names the content type it reviews and the stages an entry moves through,
each with the role that may act on it.

<div class="paths">
<div data-path="console">

1. Open **Reviews** and choose the **Definitions** tab, then **New definition**.
2. Enter `Trip review` in **Name**. Leave **Slug** empty to make one from the name.
3. Pick **Trip** in **Content type**.
4. Under **Stages, in order**, enter `Fact check` in **Stage name**, `editor` in **Required
   role** and `24` in **SLA, hours**. Choose **New stage** to add another.
5. Keep **Publish on approval** on, so the last approval publishes the trip, and choose
   **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/review/definitions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Trip review",
        "slug": "trip",
        "content_schema": "trip",
        "stages": [
          {"name": "Fact check", "required_role": "editor", "sla_duration_seconds": 86400}
        ]
      }'
```

The answer carries the definition's `id`. Keep it:

```bash
export DEF="<the id from the answer>"
```

`DELETE /api/admin/review/definitions/{id}` removes one.

</div>
</div>

![A review definition is created: the name and content type are set, a Fact check stage is given the editor role and a 24 hour SLA, and Create adds it to the list.](./media/reviews-and-releases-definition.webm)

Each stage also holds **Approvals needed**, **Escalate to** and a **Condition field**. On a free
install those controls are turned off and marked as not enabled on this engine, and a second
definition is refused with the limit it reached. With the license, they take values.

![The stage settings on a free install: Approvals needed and escalation are turned off and marked not enabled.](./media/reviews-and-releases-stage-free.png)

![The stage settings with a license: Approvals needed is set to 2 and a user is chosen to escalate to.](./media/reviews-and-releases-stage-licensed.png)

## Send a trip through review

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Unpublish **Portugal by train** from **Content > Trip**, so it is a draft again.
2. In the review bar of the entry, pick **Trip review** in **Definition** and yourself in
   **Assignee**, then choose **Start review**.
3. Open **Reviews**. The **Review assignments** list shows the trip at the **Fact check**
   stage, with its assignee and when it is due.
4. Back on the trip, write a **Comment** and choose **Approve**. The last approval publishes the
   trip. **Request changes** and **Reject** send it back instead.

</div>
<div data-path="api">

Your user id is the `id` that `GET /api/admin/auth/me` answers.

```bash
curl -X POST http://localhost:3001/api/admin/review/assignments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"entry_id\": \"$TRIP_ID\", \"definition_id\": \"$DEF\", \"assignee_id\": \"<your user id>\"}"
```

Keep the assignment's `id` from the answer, then approve it:

```bash
curl -X POST http://localhost:3001/api/admin/review/assignments/<assignment id>/transition \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"action": "approve", "comment": "Dates checked."}'
```

`GET /api/admin/review/assignments` lists every assignment.

</div>
</div>

![A review starts from the trip's review bar, the trip appears under Review assignments, and Approve with a comment publishes it.](./media/reviews-and-releases-review.webm)

The **Status** filter on **Reviews** narrows the list, and a late assignment reads **Overdue**.
[Editorial review](../features/editorial-review.md) covers who may act, deadlines, events and
reviews in a flow.

## Build a release

> **Requires** a license with the `content-pro` feature.

A release groups entries and publishes or unpublishes them together. If one fails, every entry
is put back. Write a second trip first, `Danube by bike`, and leave it as a draft.

<div class="paths">
<div data-path="console">

1. Open **Delivery > Releases** and choose **New release**.
2. Enter `Spring trips` in **Name** and a **Description**, then choose **Create**. The release
   opens.
3. Choose **Include entry**. Pick **Portugal by train** in **Entry**, keep **Publish it** under
   **When the release publishes**, and choose **Include**. Include **Danube by bike** the same
   way.
4. You can also add an entry from its editor with **Add to release**.

</div>
<div data-path="api">

Keep the id of the second trip in `TRIP2_ID`, then:

```bash
curl -X POST http://localhost:3001/api/admin/content/releases \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Spring trips", "items": [{"entry_id": "'"$TRIP_ID"'", "action": "publish"}]}'

export RELEASE_ID="<the id from the answer>"

curl -X POST http://localhost:3001/api/admin/content/releases/$RELEASE_ID/items \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entry_id": "'"$TRIP2_ID"'", "action": "publish"}'
```

`GET /api/admin/content/releases` lists the releases.

</div>
</div>

![A release named Spring trips is created, and two trips are included in it, each to be published.](./media/reviews-and-releases-build-release.webm)

On a free install, **Releases** says the feature is not enabled on this engine and links to
how to enable it, and the entry editor has no **Add to release** card. With the license, the
list and the editor show both.

![Releases on a free install: a notice that the feature is not enabled on this engine.](./media/reviews-and-releases-releases-free.png)

![Releases with a license: the Spring trips release with its two entries and its status.](./media/reviews-and-releases-releases-licensed.png)

## Schedule it or publish it now

> **Requires** a license with the `content-pro` feature.

<div class="paths">
<div data-path="console">

1. On the release, set **Publish at** and choose **Schedule**. The release reads
   **Goes out** with the time, and **Unschedule** takes it back.
2. To ship it at once instead, choose **Publish now**. Each entry shows its **Outcome**.
3. If another release changes the same entry, **Conflicts** lists it with a link to **Open
   the other release**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/content/releases/$RELEASE_ID/schedule \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"publish_at": "2027-04-01T09:00:00Z"}'

curl http://localhost:3001/api/admin/content/releases/$RELEASE_ID/conflicts \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/content/releases/$RELEASE_ID/publish \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![A publish time is set and Schedule marks the release with when it goes out, then Publish now publishes both trips and each shows its outcome.](./media/reviews-and-releases-publish.webm)

The **Status** filter on **Releases** shows **Draft**, **Scheduled**, **Published**, **Failed**
or **Canceled** releases. [Content releases](../features/content-releases.md) explains
conflicts and what happens when a license lapses.

## Next

- [7. Search](search.md): let editors and visitors find trips.
- [Editorial review](../features/editorial-review.md): deadlines, quorum, escalation and
  conditions.
- [Content releases](../features/content-releases.md): the release routes in full.
