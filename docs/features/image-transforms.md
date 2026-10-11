---
title: Image transforms
description: "Resize, crop around a focal point and convert images to WebP, AVIF, JPEG or PNG from a signed URL, generated on the first request and cached."
sidebar:
  order: 43
capability_of: media
---

> **Included free** on every install, up to 1,000 stored variants per tenant. A license with the `media-pro` feature lifts that ceiling. See [pricing](https://lyeve.com/pricing).

Every image in the [media library](media.md) gets three fixed thumbnails at
upload. Image transforms add any size, crop and format, asked for in a URL and
generated the first time someone loads it. After that the same URL is served
from storage. The URL carries a signature, so nobody can make new sizes by
editing it.

```text
/api/v1/media/{id}/transform?w=800&h=450&fit=cover&fmt=webp&q=75&sig=...
```

:::caution[Beta]
This capability is in beta. It works as this page describes, and its parameters
can still change before it is marked stable.
:::

:::tip[Best for]
- **Responsive images.** Serve one upload at the widths a `srcset` needs, each
  generated once.
- **Modern formats.** Turn a PNG or JPEG upload into WebP or AVIF without
  storing each copy yourself.
- **Crops that keep the subject.** Set a focal point on the file, and every
  `cover` crop centers on it.
- **Showing a private image to someone without an account.** A signed URL to a
  private file works until it expires.
:::

:::note[Use something else when]
- You need the original file at a stable public link. Publish it from the
  [media library](media.md), which serves it at `public_url`.
- You serve images through a CDN that resizes them itself. Point the CDN at the
  original, and it does the work instead.
:::

## How it works

| Step | What happens |
|---|---|
| Sign | An admin asks for a signed URL for one image and one set of parameters. Each tenant signs with its own key. |
| First load | The variant is generated, stored in your [object storage](object-storage.md) and served with `X-Transform-Cache: miss`. |
| Later loads | The stored variant is served with `X-Transform-Cache: hit` and the same `ETag`. |
| Delete | Deleting the file deletes its variants. |

A published file's URL does not expire unless you ask it to. A private file's
URL always expires, after an hour by default and a week at most. Responses carry
`Cache-Control`:

- `public, max-age=300` for a published file
- `private` for any other file, for an hour at most and never longer than the
  signature lasts

## Try it

You need an admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md) shows how to get one. This works
on a free install.

1. Upload an image:

   ```bash
   curl -X POST http://localhost:3001/api/admin/media \
     -H "Authorization: Bearer $TOKEN" \
     -F "file=@hero.png"
   ```

   Copy the `id` from the answer into `MEDIA_ID`.

2. Sign a 400 pixel square WebP:

   ```bash
   curl -X POST http://localhost:3001/api/admin/media/$MEDIA_ID/transform-url \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"w": 400, "h": 400, "fmt": "webp"}'
   ```

   ```json
   {
     "url": "/api/v1/media/7332599e-.../transform?fmt=webp&h=400&w=400&exp=1791043674&sig=6693...",
     "expires_at": "2026-10-03T16:07:54Z"
   }
   ```

   The file is private, so the URL expires in an hour. Copy `url` into `URL`.

3. Load it on the Content API, with no credential:

   ```bash
   curl -sI "http://localhost:3002$URL"
   ```

   The answer is `200` with `Content-Type: image/webp`, an `ETag` and
   `X-Transform-Cache: miss`.

4. Load it again. The same `ETag` comes back with `X-Transform-Cache: hit`.

5. In the admin console, open the image in the **Media library**. Its
   **Preview** drawer has a focal point picker and a builder that signs a URL
   with **Width**, **Height**, **Fit**, **Format**, **Quality** and
   **Expires**.

## Parameters

| Parameter | Values | Default |
|---|---|---|
| `w`, `h` | 1 to 4096 pixels | The original's size. With one side set, the other keeps the image's shape. |
| `fit` | `cover` crops to fill the box, `contain` fits inside it, `fill` stretches to it. It applies when both `w` and `h` are set. | `cover` |
| `fmt` | `webp`, `avif`, `jpeg`, `png` | The original's format for JPEG, PNG and AVIF, and WebP for anything else |
| `q` | 1 to 100 | `80`. Ignored for PNG. |
| `focal` | `x,y`, each from 0 to 1 | The file's focal point, or the center |

An image is never enlarged. A box larger than the original shrinks to fit it and
keeps its shape. AVIF needs the `avifenc` tool on the server. Without it an AVIF
request answers WebP, the `Content-Type` says so, and that answer is not
stored.

## Sign a URL

`POST /api/admin/media/{id}/transform-url` takes the parameters above as `w`,
`h`, `fit`, `fmt`, `q` and `focal` (`{"x": 0.3, "y": 0.4}`), plus `expires_in`
in seconds, from 0 to 604800. The answer is a path and `expires_at`, which is
`null` for a URL that never expires. Put your Content API address in front of
the path.

- Signing needs `ENCRYPTION_KEY`, which protects each tenant's signing key.
- A URL signed for one tenant works on no other.
- An admin previewing an image can call `GET /api/admin/media/{id}/transform`
  with the same parameters and no signature.
- A super admin who suspects a URL leaked calls
  `POST /api/admin/media/transform-key/rotate` from a signed-in session. Every
  URL signed with the old key answers `403` from then on, and you sign new ones.

## Set a focal point

The focal point is where a `cover` crop centers. Set it on the file, free on
every install:

```bash
curl -X PATCH http://localhost:3001/api/admin/media/$MEDIA_ID \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"focal_point": {"x": 0.3, "y": 0.4}}'
```

`{"focal_point": null}` clears it back to the center. A new focal point retires
the crops cut around the old one, and a `focal` parameter in a URL overrides the
stored point for that URL.

## The free ceiling

Each tenant keeps up to 1,000 generated variants for free. Only a request that
would generate a new variant counts. Serving one already in storage is always
free, so a URL that has been loaded once keeps working past the ceiling.
Signing URLs, previews of stored variants and the focal point never count.
Deleting a file frees the places its variants held.

At the ceiling, a request for a new variant on either transform route answers
`402`. The admin preview names the ceiling and the count, so the console can
show them:

```json
{"error": "cap_exceeded", "cap": "media.variants", "limit": 1000, "current": 1000, "upgrade_url": ""}
```

The signed public URL may be loaded by a visitor with no credentials, so its
refusal says only `{"error": "payment_required"}`. A license with `media-pro`
lifts the ceiling.

## If the license lapses

Every variant already in storage keeps being served, including the ones
generated past 1,000 while licensed. Only the next new variant is refused, with
the `402` above. The original file, its public URL and its thumbnails are never
counted.

## Routes

| Method | Path | Who | Purpose |
|---|---|---|---|
| `GET`, `HEAD` | `/api/v1/media/{id}/transform` | Anyone with a signed URL | Serve a transform. |
| `POST` | `/api/admin/media/{id}/transform-url` | `admin`, `super_admin` | Sign a transform URL. |
| `GET` | `/api/admin/media/{id}/transform` | `admin`, `super_admin` | Preview a transform, unsigned. |
| `POST` | `/api/admin/media/transform-key/rotate` | `super_admin`, signed-in session | Retire every URL the tenant signed. |

An [admin token](../how-to/admin-tokens.md) with the `media:read` grant can sign
and preview.

## Errors

| Status | Message | Cause |
|---|---|---|
| `400` | `invalid transform: ...`, naming the parameter | A value out of range, a name the transform does not know, or a name given twice. |
| `400` | `expires_in must be from 0 to 604800 seconds` | The signature would last longer than a week. |
| `400` | `exp must be a Unix time` | The URL's `exp` was edited. |
| `402` | `cap_exceeded`, `media.variants` | The admin preview asked for a new variant, the tenant stores 1,000 and the license does not carry `media-pro`. Stored variants are still served. |
| `402` | `payment_required` | The same ceiling, reached through a signed public URL. The body names neither the ceiling nor the count. |
| `403` | `transform URL is not signed` | No `sig` in the URL. |
| `403` | `transform URL signature does not match` | The URL was edited, signed for another tenant, or signed with a key that was rotated. |
| `403` | `transform URL has expired` | The `exp` time has passed. Sign a new URL. |
| `404` | `not found` | No such file in the tenant, or a private file reached through a URL without an expiry. |
| `422` | `only images can be transformed` | The file is not an image. |
| `422` | Describes the image | The original is over 16384 pixels on a side, over 50 megapixels or over 64 MiB. |
| `503` | `this instance has no encryption key, so it cannot sign transforms` | Set `ENCRYPTION_KEY` before rotating the key. |

## Related

- [Media](media.md): uploads, the library, thumbnails and public links.
- [Object storage](object-storage.md): where variants are stored.
- [Caching](caching.md): caching content reads.
