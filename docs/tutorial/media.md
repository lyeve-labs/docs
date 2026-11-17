---
title: "5. Media"
description: "Upload a photo to the media library, describe and publish it, use it as a trip's cover, import a file from a URL, crop around a focal point and choose where files are stored."
sidebar:
  order: 5
covers:
  console:
    - "/admin/media"
    - "/admin/settings/storage"
  api:
    - "POST /api/admin/media"
    - "PATCH /api/admin/media/{id}"
    - "POST /api/admin/media/search"
    - "POST /api/admin/media/import"
    - "POST /api/admin/media/{id}/transform-url"
    - "DELETE /api/admin/media/{id}"
    - "PUT /api/v1/content/{schema}/{id}"
    - "GET /api/admin/storage/providers"
    - "POST /api/admin/storage/providers"
    - "POST /api/admin/storage/providers/{id}/test"
verified:
  engine: v0.62.0
  console: v0.28.0
---

In this lesson you give the trip from [lesson 4](write-and-publish.md) a cover photo. You
upload it, describe it for readers who cannot see it, publish it and link it from the trip. You
need the trip in `TRIP_ID`, a token in `TOKEN` for the API steps, and a photo on your computer,
here `lisbon.jpg`.

## Upload a photo

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Open **Media library** in the sidebar. A new library says **No files yet**.
2. Choose **Upload files** and pick `lisbon.jpg`. You can pick several files at once.
3. The file appears in the library. **View** switches between a grid and a table.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/media \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@lisbon.jpg" \
  -F "folder=/trips" \
  -F "alt_text=Yellow tram on a steep street in Lisbon"
```

The answer carries the file's `id`. Keep it:

```bash
export MEDIA_ID="<the id from the answer>"
```

</div>
</div>

![Upload files is chosen, a photo is picked, and it appears in the media library grid.](./media/media-upload.webm)

## Describe and publish it

> **Included free** on every install.

A file is private until you publish it. Only someone signed in to the console can open a
private file, and a content field can link only a published one.

<div class="paths">
<div data-path="console">

1. Choose the photo. A panel opens with its preview and **Details**.
2. Write what the photo shows in **Alt text**, add `lisbon, tram` in **Tags** and `/trips` in
   **Folder**, then choose **Save**.
3. Choose **Publish**. The badge changes from **Private** to **Published**.
4. Copy the address with **Copy URL**, or as Markdown or HTML with the buttons below it.

</div>
<div data-path="api">

```bash
curl -X PATCH http://localhost:3001/api/admin/media/$MEDIA_ID \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"alt_text": "Yellow tram on a steep street in Lisbon", "tags": ["lisbon", "tram"], "public": true}'
```

The answer carries the file's `public_url`, which a site or a content field uses.

</div>
</div>

![A photo is opened, its alt text, tags and folder are saved, and Publish turns the badge from Private to Published.](./media/media-describe-publish.webm)

With the `ai` feature licensed and a model provider connected in lesson 21, **Suggest alt text**
asks a model to describe the image. Review the text before you save it.

## Use it as the trip's cover

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Open **Portugal by train** from **Content > Trip**.
2. In **Cover**, choose **Choose from library**. **Choose a file** lists the published files.
3. Pick the photo, then choose **Save**.

</div>
<div data-path="api">

A `media` field holds the file's public link:

```bash
curl -X PUT http://localhost:3002/api/v1/content/trip/$TRIP_ID \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"cover": "<the public_url of the photo>"}}'
```

</div>
</div>

![The trip's Cover field opens Choose a file, the Lisbon photo is picked, and the trip is saved.](./media/media-cover.webm)

## Import a file from a URL

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. In **Media library**, choose **Import from URL**.
2. Enter the file's address in **Address**, and its **Alt text**.
3. Tick **Publish it, so a site or a content field can use its address** if you want it public
   at once, then choose **Import**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/media/import \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/porto.jpg", "folder": "/trips", "alt_text": "Porto from the river", "public": true}'
```

</div>
</div>

The engine copies the file into the library. It refuses private and internal addresses.

## Find a file

> **Included free** on every install.

<div class="paths">
<div data-path="console">

Type in **Filter media** above the library. It matches the file name.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/media/search \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "lisbon", "kind": "image"}'
```

</div>
</div>

## Crop around a focal point and resize

> **Included free** on every install, up to 1,000 stored image variants per tenant. More need a
> license with the `media-pro` feature.

A focal point tells every crop where the subject is. A transform URL asks for a resized or
converted copy, made on the first request and cached.

<div class="paths">
<div data-path="console">

1. Open the photo. Under **Focal point**, click the tram, then choose **Save focal point**.
2. Under **Transform URL**, enter a **Width** of `800`, pick a **Format** such as WebP, and keep
   **Center a crop on the focal point** ticked.
3. Choose **Build URL** and copy the **Signed URL**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/media/$MEDIA_ID/transform-url \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"w": 800, "fmt": "webp"}'
```

The answer is a path and `expires_at`. Put `http://localhost:3002` in front of the path.

</div>
</div>

![A focal point is clicked on the tram and saved, then a WebP transform 800 pixels wide is built and its signed URL appears.](./media/media-focal-transform.webm)

[Image transforms](../features/image-transforms.md) lists every parameter, how long a URL lasts
and what happens at the free ceiling. To remove a file, choose **Delete** in its panel, or
`DELETE /api/admin/media/{id}`. A page that links it can no longer load it.

## Choose where files are stored

> **Included free** on every install, S3, MinIO, Google Cloud Storage and Azure Blob included.

With no provider, uploads go to a directory on the engine host, which suits one engine. A
provider sends them to an object store instead. Only a super admin adds or changes one.

| Part | What it does |
|---|---|
| **New uploads go to** | Names the provider that receives the next upload |
| **Provider**, **Writes to**, **State**, **Priority** | Each provider, where it writes and whether it receives uploads |
| **Type**, **Name** | The kind of store, and the name you know it by |
| **Priority** | Which enabled provider receives uploads first |
| **Use TLS**, **Path-style addresses**, **Unsigned public URLs** | How the engine reaches the store and links published files |
| **Enabled** | Whether the provider receives uploads |
| **How storage works** | The fallback directory and how tenants share providers |

<div class="paths">
<div data-path="console">

1. Open **Settings > Storage** and choose **New provider**.
2. Pick the **Type**, such as S3, enter a **Name** and the bucket and credentials the form asks
   for.
3. Leave **Enabled** off and save. Then test the provider from its row. A provider with wrong
   credentials fails every upload it receives.
4. Once the test passes, turn on **Enabled**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/storage/providers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "trip-photos",
        "provider_type": "s3",
        "bucket": "<your bucket>",
        "region": "eu-west-1",
        "access_key": "<access key id>",
        "secret_key": "<secret access key>",
        "use_ssl": true,
        "enabled": false
      }'

curl -X POST http://localhost:3001/api/admin/storage/providers/<provider id>/test \
  -H "Authorization: Bearer $TOKEN"
```

`GET /api/admin/storage/providers` lists them. Credentials are stored encrypted and never shown
again.

</div>
</div>

![A new S3 provider is filled in and saved disabled, the test from its row passes, and it is enabled to receive uploads.](./media/media-storage-provider.webm)

[Storage](../features/object-storage.md) covers every store, uploads straight from the client
and the settings behind it.

## Next

- [6. Reviews and releases](reviews-and-releases.md): send the trip through review and publish
  several entries at once.
- [Media](../features/media.md): the library in full, with virus scanning and public links.
- [Media library for editors](../guides/user/media-library.md): the same screens for people who
  write.
