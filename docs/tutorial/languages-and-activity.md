---
title: "8. Languages and activity"
description: "Add a second language to the travel guide, translate a trip, read it in the reader's language, and see who changed which entry in the content activity log."
sidebar:
  order: 8
covers:
  console:
    - "/admin/settings/localization"
    - "/admin/content-activity"
  api:
    - "GET /api/admin/localization/locales"
    - "PUT /api/admin/localization/locales"
    - "POST /api/admin/content/{id}/translations"
    - "GET /api/v1/content/{schema}/{id}"
    - "GET /api/admin/content/activity"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson the travel guide learns Portuguese, and you read back who changed each entry. You
need the trip from [lesson 4](write-and-publish.md) in `TRIP_ID`, with `title` and `summary`
marked as translated as [lesson 3](model-content.md) did, and a token in `TOKEN` for the API
steps.

## Add a second language

> **Included free** on every install, with two locales per tenant, the default one included.
> More locales need a license with the `localization-pro` feature.

<div class="paths">
<div data-path="console">

1. Open **Settings > Locales**. Keep `en` as the **Default locale**.
2. Under **Enabled locales**, pick `pt` in **Add locale** and choose **Add locale**. Each
   locale adds a tab to every entry's **Translations** card.
3. Under **Fallback chain**, pick `pt` in **Append a locale** and choose **Append** if readers
   of a regional variant, such as `pt-BR`, should fall back to it.
4. Choose **Save**. The page says **Locales saved.**

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/localization/locales \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"default_locale": "en", "enabled_locales": ["en", "pt"], "fallback_chain": []}'
```

`GET` on the same route reads the settings back.

</div>
</div>

![The Locales settings: pt is added under Enabled locales and saved.](./media/languages-and-activity-locales.mp4)

A third locale is where the editions part. On a free install, adding it is refused with
**No more locales on this engine** and how many are in use, and the API answers `402` with
`"cap": "localization.locales"`. With the license, it saves like the second.

![A third locale on a free install: the save is refused with No more locales on this engine, 2 of 2 in use.](./media/languages-and-activity-third-locale-free.png)

![A third locale with a license: es is added beside en and pt and saved.](./media/languages-and-activity-third-locale-licensed.png)

## Translate a trip

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Open **Portugal by train** from **Content > Trip**.
2. In the **Translations** card, choose the `pt` tab. Its status reads **Not started**, and
   readers of `pt` see the source until you save.
3. Enter `Portugal de comboio` in **Title** and a Portuguese summary in **Summary**, then choose
   **Save**. The status reads **Draft**.
4. Choose **Mark translated** once the text is checked. The status reads **Translated**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/content/$TRIP_ID/translations \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "locale": "pt",
        "title": "Portugal de comboio",
        "body": {"title": "Portugal de comboio", "summary": "<p>De Lisboa ao Porto numa semana.</p>"},
        "translation_status": "translated"
      }'
```

`PUT /api/admin/content/{id}/translations/{locale}` changes only the fields you send.

</div>
</div>

![The pt tab of the Translations card: the title and summary are typed, saved as a draft, then marked translated.](./media/languages-and-activity-translate.mp4)

When someone changes the English source later, the `pt` tab reads **outdated**, so you know
which languages need another look.

## Read the trip in a language

> **Included free** on every install.

This is an API task: a site asks for the reader's language and the engine answers with the
translation, or the next locale in the fallback chain, or the source.

```bash
curl "http://localhost:3002/api/v1/content/trip/$TRIP_ID?locale=pt" \
  -H "Authorization: Bearer $TOKEN"
```

Without `locale`, the `Accept-Language` header decides. [Localization](../features/localization.md)
covers the fallback chain, completeness and bulk export for a translation agency.

## See who changed what

> **Included free** on every install.

Content activity lists each create, change, publish and delete of an entry, with who did it and
through what. It records while **Record who changed which entry** is on, which lesson 4 kept.

| Part | What it does |
|---|---|
| **Content type**, **Action**, **Source**, **From** | Narrow the list, then **Apply**. **Clear filters** shows everything again |
| **When**, **Who**, **Entry**, **Type** | Each change and who made it |
| **Changed fields**, **Version** | The fields the change touched, and the version it made |
| **API key**, **through an admin token** | Marks a change made by a script rather than a person |

<div class="paths">
<div data-path="console">

1. Open **Insight > Content activity**. The translation and the edits of lessons 4 to 6 are
   listed, newest first.
2. Pick **Trip** in **Content type** and choose **Apply**.
3. Choose a person to show only their changes, or an entry to show only its changes. Open the
   entry from its row.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/content/activity?schema=trip&limit=20" \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![Content activity is narrowed to Trip, then to changes by one person, and an entry is opened from its row.](./media/languages-and-activity-activity.mp4)

A super admin of an instance with several tenants also sees a **Tenant** filter. The audit log
in lesson 18 records every other action on the instance.

## Next

- [9. Import, export and config sync](import-and-export.md): load stops from a file and move
  the setup to another engine.
- [Localization](../features/localization.md): locales, fallback and translations in full.
- [Content history](../features/content-history.md#activity-log): what the activity log keeps
  and for how long.
