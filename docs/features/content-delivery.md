---
title: Public content delivery
description: Serve the published entries of chosen content types to anyone without a credential, in answers a CDN and a browser may cache, and purge the CDN when content changes.
capability_of: content
---

> **Included free** on every install.

Public content delivery serves the published entries of the content types you
mark to anyone, with no token, API key or session. Its answers depend only on
the host and the URL, so a CDN in front of the API can keep them and answer
your visitors without reaching the instance.

The Content API's own reads at `/api/v1/content/` take a credential and are
always answered `private, no-store`, so no shared cache keeps them. Use the
delivery routes for content meant for everyone, and keep the Content API for
drafts, closed types and callers who sign in.

:::tip[Best for]
- **A public site or app behind a CDN.** Pages, posts and product data that
  every visitor sees the same way.
- **Reading content from the browser.** Any site may read a delivery answer,
  with no token in the page.
:::

:::note[Use something else when]
- A reader must sign in, or sees drafts. Use the
  [Content API](../reference/api-endpoints.md#read-and-write-content) with a
  credential.
- You need sorting, search or a language other than the stored one. The
  delivery routes take none of these in this release.
:::

## How it works

| Step | What happens |
|---|---|
| Mark | A content type's definition carries `"public_delivery": true`. Unmarked is the default, so nothing becomes public on upgrade. |
| Resolve | The tenant comes from the request's host, as for every public route. Nothing in the request can name another tenant. |
| Refuse | A request that carries a bearer token, an API key or a session cookie answers `400`. |
| Read | Only published entries are served. A field marked `"private": true` is left out, and so is a relation into a type that is not marked. |
| Answer | The answer is public, with a lifetime for browsers, a longer one for shared caches and a strong `ETag`. |

## Try it

You need an admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md#3-sign-in) shows how to get one.
The example runs on a single-tenant install on `localhost`, where every
request belongs to the one tenant.

**1. Create a marked content type.** The Schema builder in the console has no
switch for drafts and publishing, so this step is an API call. To mark a type
you already have, see [Mark a content type](#mark-a-content-type). Save this
definition:

```bash
curl -X POST http://localhost:3001/api/admin/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "press_release",
        "display_name": "Press release",
        "with_draft_publish": true,
        "public_delivery": true,
        "fields": [
          { "name": "title", "field_type": "text", "required": true },
          { "name": "slug", "field_type": "text", "indexed": true },
          { "name": "body", "field_type": "rich_text" },
          { "name": "embargo_note", "field_type": "text", "private": true }
        ]
      }'
```

The answer is `200` with the stored definition. `embargo_note` is a field your
editors see in the console and on the Content API, and the delivery routes
never serve. Neither mark changes the database tables.

**2. Write and publish an entry.**

```bash
curl -X POST http://localhost:3002/api/v1/content/press_release \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"title": "Spring launch", "slug": "spring-launch", "body": "<p>Out now.</p>", "embargo_note": "Hold until 9:00"}}'
```

Copy the `id` of the answer into `ID`, then publish the entry:

```bash
curl -X PUT http://localhost:3002/api/v1/content/press_release/$ID/publish \
  -H "Authorization: Bearer $TOKEN"
```

**3. Read it with no credential.**

```bash
curl -i http://localhost:3002/api/v1/delivery/press_release/$ID
```

The answer is `200` with the entry, without `embargo_note`:

```text
HTTP/1.1 200 OK
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=60, stale-if-error=600
Etag: "<sha-256 of the body>"
Access-Control-Allow-Origin: *
```

The list is at `/api/v1/delivery/press_release` and answers a bare array of
the published entries. A draft is not in the list, and a read of a draft by id
answers `404`, as an id that does not exist does.

**4. Revalidate.** Send the `ETag` back:

```bash
curl -i http://localhost:3002/api/v1/delivery/press_release/$ID \
  -H 'If-None-Match: "<the ETag>"'
```

The answer is `304` with no body while the entry is unchanged.

## Mark a content type

<div class="paths">
<div data-path="console">

1. Open **Schema builder** and choose the content type.
2. Open **Table settings** and tick **Serve on public delivery**. A type with no
   draft and publish shows a warning, because every one of its entries is then
   served.
3. In the row of each field the delivery routes must never serve, tick
   **Private**.
4. Choose **Save**.

</div>
<div data-path="api">

Send the type's whole definition again to `POST /api/admin/schemas`, as step 1
of [Try it](#try-it) does, with `"public_delivery": true` added and
`"private": true` on each field the delivery routes must never serve.

</div>
</div>

After you turn the mark off, a CDN or a cache rule keeps the answers it already
holds until they expire or are purged.

## What is served

- **Published entries only.** On a type with draft and publish, an entry is
  served once its status is `published`, and a draft, an archived or a deleted
  entry is not. On a type without draft and publish, every entry is served.
- **No private fields.** A field marked `"private": true` is never in an
  answer, and a filter on it answers as one on a field that does not exist.
  The Content API still serves the field to callers who may read it.
- **Relations into marked types only.** A relation into a type that is not
  marked is left out of the entry. A related entry that is not published reads
  as `null`.
- **The tenant from the host.** On a multi-tenant install a visitor reaches a
  tenant through a [domain mapped to it](tenants.md#serve-a-tenant-on-its-own-domain).
  A host that resolves to no tenant answers `404`, and so does a type the
  tenant has not marked.

## The URL grammar

Each entry and each list has exactly one URL, so a cache holds one copy of
each answer. The routes take these parameters and nothing else:

| Parameter | On | Does |
|---|---|---|
| `limit` | The list | Entries per page, 1 to 200. Default 25. |
| `offset` | The list | Entries to skip, 0 to 10000. Default 0. |
| `filters[<field>]=<value>` | The list | Keep entries whose field equals the value. At most 4 filters, each value at most 255 characters. |
| `populate` | The list and the read by id | Relations to return inline, comma-separated, at most 8 paths. |
| `depth` | The list and the read by id | Levels of relations to return inline, 0 to 2. |
| `fields` | The list and the read by id | Return only the named fields of each entry, comma-separated, such as `fields=title,slug`. |

- **Filters** work on `id` and on fields of the types `text`, `email`, `url`,
  `uid`, `relation`, `integer`, `decimal`, `boolean`, `date` and `datetime`.
  The value must be in the field's own form: an id or a relation as a UUID in
  lower case with hyphens, `true` or `false` for a boolean, `2026-11-04` for a
  date. Filter a `belongs_to` relation by its column, such as
  `filters[author_id]=<uuid>`. Use `curl -g` so curl does not read the
  brackets itself.
- **Populate and depth** follow `belongs_to` relations into marked types only.
  A path has at most two segments, such as `author` or `author.company`, and
  each segment names a relation. A wildcard, a field list, a to-many relation
  or a path into a type that is not marked answers `400`. `depth` skips
  to-many relations.
- **Fields** name each field once, with no space around a name. `id`,
  `created_at` and `updated_at` are in every entry and cannot be named. A name
  the type does not have, or a private field, answers `400`.
- **An id** must be written in lower case with hyphens. Any other spelling,
  such as upper case or braces, answers `404`.
- **The type's name** must be spelled exactly as it is stored. Another
  spelling answers `404`.
- **No language.** `locale` is not accepted in this release, and
  `Accept-Language` is not read. An entry is served as it is stored.

A parameter not in the table, a parameter sent twice, or a value out of its
range answers `400`. Nothing is clamped or ignored, because each would be a
second URL for the same answer.

```bash
curl -g "http://localhost:3002/api/v1/delivery/press_release?limit=10&filters[slug]=spring-launch"
```

## The headers

| Header | Value |
|---|---|
| `Cache-Control` | `public, max-age=60, s-maxage=300, stale-while-revalidate=60, stale-if-error=600` |
| `ETag` | A strong tag, the SHA-256 of the body. A matching `If-None-Match` gets `304`. |
| `Access-Control-Allow-Origin` | `*`, whatever `CORS_ORIGINS` names. Credentials are never allowed. |

- `max-age` is how long a browser keeps an answer. Set it with
  `DELIVERY_MAX_AGE`.
- `s-maxage` is how long a CDN or another shared cache keeps it. Set it with
  `DELIVERY_SHARED_MAX_AGE`.
- `stale-while-revalidate` and `stale-if-error` are fixed. A cache may serve
  an answer 60 seconds past its lifetime while it fetches a new one, and up to
  10 minutes past it while the instance fails.

Both settings take whole seconds from 1 to 86400 and are read once at boot.
See [configuration](../reference/configuration.md#caching-and-redis).

A preflight to a delivery path is answered `204` for any origin, allowing
`GET` and `HEAD` with the `If-None-Match` and `If-Modified-Since` headers, and
the browser may keep it for a day.

An error answer (`400`, `404`, `429` or `503`) is `no-store`, so no cache
keeps it, and an entry shows the moment it is published.

### Fetch from a browser

A browser sends no cookie on a cross-origin fetch by default. On the same
origin as the admin console it sends the session cookie, and the request
answers `400`. Fetch with `credentials: "omit"` there:

```js
const res = await fetch("https://api.example.com/api/v1/delivery/press_release?limit=10", {
  credentials: "omit",
});
const releases = await res.json();
```

## Put a CDN in front

1. **Forward the host.** The instance picks the tenant from the `Host` the
   request arrives with. Configure the CDN to send the visitor's host to the
   origin, or the host of a domain mapped to the tenant, not the origin's own
   address. A host that resolves to no tenant answers `404`.
2. **Send no credential.** Strip `Authorization`, `X-API-Key` and cookies on
   the delivery paths, or a request answers `400`.
3. **Name the CDN's addresses.** Behind a CDN the instance sees the CDN's
   addresses. Set `TRUSTED_PROXIES` to the CDN's published ranges, so the
   rate limit counts each visitor by the address in `X-Forwarded-For`.
   Without it, every visitor behind one edge shares one limit, and a busy
   edge gets `429`.
4. **Purge after a change.** Set up the purge below, or accept the delay
   without one.

The delivery routes run on the general per-address limit for public routes.
`PUBLIC_RATE_LIMITS` gives them a limit of their own, with entries for
`GET:/api/v1/delivery/{schema}` and `GET:/api/v1/delivery/{schema}/{id}`.

### Without a purge

A change shows within `s-maxage` plus `stale-while-revalidate`, six minutes
with the defaults. A browser that already holds an answer keeps it for its
own `max-age` on top, because no purge reaches a browser. If that delay is
acceptable, skip the purge.

### Purge with a webhook

A [webhook](webhooks.md) is free, and it can call a purge endpoint you run,
such as a small function at your CDN that checks the signature and purges the
delivery prefix with your CDN's own API.

```bash
curl -X POST http://localhost:3001/api/admin/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Purge delivery cache",
        "url": "https://purge.example.com/lyeve",
        "events": ["after_create", "after_update", "after_delete"],
        "secret": "replace-with-a-long-random-secret"
      }'
```

With no `schemas`, the webhook fires for every content type and also when a
type's definition changes, with `"schema": "sys_schemas"` and the type's name
in `data`. Creating a webhook needs the `super_admin` role. Your endpoint
receives the body and headers [webhooks](webhooks.md) describes, and decides
what to purge.

## Response cache on the instance

The [response cache](caching.md#cache-the-answers-of-public-routes) can keep
delivery answers on the instance too, so a request that reaches the origin
skips the database. One rule on `/api/v1/delivery/**` covers every type.

## Errors

| Status | When |
|---|---|
| `400` | The request carries a credential, a parameter is not in the grammar or is sent twice, a value is out of range, or a filter or a populate path is refused. |
| `404` | No such entry, an entry that is not published, an id not in lower case with hyphens, a type that is not marked, or a host that resolves to no tenant. |
| `429` | Too many requests from one address. |
| `503` | The type's definition or a related entry could not be read. Try again. |

A request with a credential answers
`the delivery routes take no credential, so send the request without one`.

## Related

- [API endpoints](../reference/api-endpoints.md): the Content API with a
  credential.
- [Caching](caching.md): the response cache on the instance.
- [Tenants](tenants.md): serve a tenant on its own domain.
