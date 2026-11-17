---
title: "7. Search"
description: "Search the travel guide from LyEve Console, see what readers search for, make one word match another, weigh title matches, and let visitors search trips with no credential."
sidebar:
  order: 7
covers:
  console:
    - "/admin/search"
    - "/admin/search/synonyms"
    - "/admin/search/settings"
  api:
    - "GET /api/admin/search"
    - "GET /api/admin/search/analytics"
    - "POST /api/admin/search/synonyms"
    - "GET /api/admin/search/synonyms"
    - "DELETE /api/admin/search/synonyms/{id}"
    - "GET /api/admin/search/public-schemas"
    - "PUT /api/admin/search/public-schemas"
    - "PUT /api/admin/search/ranking"
    - "POST /api/admin/search/reindex"
    - "GET /api/v1/search/{schema}"
verified:
  engine: v0.62.0
  console: v0.28.0
---

> **Requires** a license with the `search` feature.

In this lesson you search the trips and stops you wrote, teach search that two words mean the
same, and open trip search to visitors of your site. You need the entries from
[lesson 4](write-and-publish.md), a license that includes search, and a token in `TOKEN` for
the API steps.

**Search** has three tabs: **Search**, **Synonyms** and **Settings**. On a free install the
screen says search is not enabled on this engine and links to how to enable it, and the Content
API's search route answers `404`. With the license, the tabs work as below.

![Search on a free install: a notice that search is not enabled on this engine.](./media/search-free.png)

![Search with a license: a query for Lisbon lists the trip and the stop with their rank.](./media/search-licensed.mp4)

## Search the content

Entries are indexed when they are saved, so the trips and stops are already searchable.

<div class="paths">
<div data-path="console">

1. Open **Search** in the sidebar.
2. Type `lisbon` in **Search content** and choose **Search**.
3. Each hit shows its title, content type, status, slug and rank.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/search?q=lisbon&limit=20" \
  -H "Authorization: Bearer $TOKEN"
```

The answer holds `results`, each with its rank, and `total`, the number of matches in all.

</div>
</div>

A search that finds nothing says so and suggests a broader term.

## See what readers search for

<div class="paths">
<div data-path="console">

**Search analytics**, under the results, shows the **Searches** of the **Last 30 days** and the
ones that returned **No results**. A term people look for and never find is a page you have not
written yet.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/search/analytics \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

## Make one word match another

A visitor who searches `railway` should find the trip that says `train`. A synonym group makes a
search for one term also match the others in its group.

<div class="paths">
<div data-path="console">

1. Open the **Synonyms** tab and choose **New synonym**.
2. Enter `train` in **Term** and `railway, rail` in **Also matches**.
3. Choose **Create**. The group appears in **Synonyms**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/search/synonyms \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Train", "base_term": "train", "synonyms": ["railway", "rail"]}'
```

`GET /api/admin/search/synonyms` lists the groups, and
`DELETE /api/admin/search/synonyms/{id}` removes one.

</div>
</div>

![New synonym opens a form, train is entered with railway and rail as matches, and Create adds the group to the list.](./media/search-synonym.mp4)

## Let visitors search trips

**Settings** decides what anyone may search on your site, with no credential.

| Part | What it does |
|---|---|
| **Searchable by anyone** | The content types a visitor may search |
| **Example GET request** | The request your site sends, ready to copy |
| **Ranking weights** | How much a match in **Title**, **Body** and **Tags** counts, for every content type without its own |
| **Rebuild index** | Indexes every entry again, which you need only after data changed outside the engine |

<div class="paths">
<div data-path="console">

1. Open the **Settings** tab.
2. Under **Public search**, pick **Trip** in **Searchable by anyone** and choose **Save**.
3. Copy the **Example GET request** to try it from your site.
4. Under **Ranking weights**, raise **Title** above **Body** so a trip named for a place ranks
   first, and choose **Save**.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/search/public-schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"schemas": ["trip"]}'

curl -X PUT http://localhost:3001/api/admin/search/ranking \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"schema_name": "*", "title_weight": 1.0, "body_weight": 0.3, "tag_weight": 0.5}'
```

A visitor's search takes no credential:

```bash
curl "http://localhost:3002/api/v1/search/trip?q=porto&limit=10"
```

`POST /api/admin/search/reindex` rebuilds the index.

</div>
</div>

![Trip is made searchable by anyone and saved, the example request is copied, and the title weight is raised above the body weight.](./media/search-settings.mp4)

A visitor sees published trips only. [Search](../features/search.md) covers instant search,
facets, Elasticsearch and Meilisearch, and the limits on public search.

## Next

- [8. Languages and activity](languages-and-activity.md): translate a trip and see who changed
  what.
- [Search](../features/search.md): every search route and setting.
- [Search and organization for editors](../guides/user/search-and-organization.md): finding
  content day to day.
