---
title: Start from a flow template
description: "Pick one of eighteen starter flows, see what each one needs, create it as a draft, test it and publish it."
sidebar:
  order: 13
---

> **Included free** on every install. Templates that call outside the instance require a license with the `flow-pro` feature. See [pricing](https://lyeve.com/pricing).

Every install ships eighteen starter flows. Each is a complete
[flow](../features/flows.md) you create as a draft, point at your own content
types and settings, test and publish. Find yours in the table, set up what it
needs, then follow the steps below.

## Choose a template

Free templates run on every install. A `pro` template needs `flow-pro`, because
it sends mail, calls an outside API or reads a sheet or a database.

| Template | What it does | Best for | Tier | What it needs |
|---|---|---|---|---|
| `join-three-tables-api` | A `GET` endpoint that returns orders with their customer and shipments, cached 30 seconds and limited to 120 calls a minute | A frontend that needs related entries in one call | free | `orders`, `customers` and `shipments` content types |
| `bulk-import` | A `POST` endpoint that takes a JSON array of rows and creates one entry per row | Loading a spreadsheet export into a content type | free | A `customers` content type with `email`, `name`, `company` and `plan` fields, or yours in its place |
| `seo-sitemap` | Answers `sitemap.xml` built from the `pages` entries marked published | Search engines on a site built from LyEve content | free | A `pages` content type, variable `site_url` |
| `seo-robots` | Answers `robots.txt` built from rules kept as entries | Letting editors change crawl rules without a deploy | free | A `robots_rules` content type, variable `sitemap_url` |
| `edge-geo-redirect` | Redirects a visitor to a regional page by the country header your CDN sends | Sending visitors to their country's site | free | Nothing |
| `scheduled-import` | Fetches a JSON array of rows from an address every hour and creates or updates one entry per row | Keeping a content type in step with a feed another system publishes | pro | A `products` content type with `sku`, `name` and `price` fields, or yours in its place, variable `import_feed_url`, secret variable `import_feed_token` |
| `nightly-sheet-export` | Appends paid orders to a Google Sheet at 02:00 UTC | A finance team that works in Sheets | pro | A `google_sheets` datasource `reports`, variable `orders_sheet_id` |
| `enrich-on-create` | Looks up a new customer's email domain and writes company details back | Filling in company, industry and size without typing | pro | Secret variable `enrichment_api_key`, your provider's URL |
| `notify-on-failure` | Emails the operator when any flow run fails | Hearing about broken automations | pro | Variable `ops_email`, an email sender |
| `cdn-purge` | Purges an updated entry's two pages from Cloudflare | A cached site that must show edits at once | pro | An `http` datasource `cloudflare`, variables `cloudflare_zone_id` and `site_url` |
| `chat-notify` | Posts each new article to Slack, Discord or Telegram | Telling a team channel what was published | pro | An `http` datasource `chat`, variable `chat_provider` |
| `review-workflow` | Moves a `reviews` entry through draft, in review, approved and rejected, and mails the next person | A small approval loop | pro | A `reviews` content type, an email sender |
| `comment-moderation` | Asks a moderation service whether a new comment is spam and sets its status | Keeping spam off public comments | pro | The `comments` preset, an `http` datasource `moderation` |
| `form-submitted` | Mails each form submission to the form's address and can post it to a webhook | Contact and sign-up forms | pro | The `forms` preset, an email sender, and for the webhook an `http` datasource `form_webhook` and the variable `form_webhook` set to `on` |
| `crm-hubspot-contacts` | Creates or updates a HubSpot contact for each new customer, matched by email | Keeping HubSpot in step with sign-ups | pro | Secret variable `hubspot_token` |
| `crm-pipedrive-deals` | Creates a Pipedrive deal for each new order and stores the deal id on it | Sales follow-up on orders | pro | Secret variable `pipedrive_token`, a `pipedrive_deal_id` field on orders |
| `crm-salesforce-contacts` | Creates a Salesforce contact for each new customer | Teams that sell from Salesforce | pro | Variable `salesforce_instance_url`, secret variable `salesforce_token` |
| `cost-ledger` | Writes a nightly ledger of API requests per key and AI spend per model | Charging usage back to the teams behind each key | pro | A `postgres`, `mysql` or `mssql` datasource `replica`, variables `tenant_id` and `price_per_1k_requests`, a `cost_entry` content type |

On an instance without `flow-pro`, a pro template is still listed, and saving it
answers `402` naming each node that needs the license. See
[free and flow-pro](../features/flows.md#free-and-flow-pro).

## Before you start

- An admin token in `TOKEN`. The [quickstart](../getting-started/quickstart.md)
  shows how to get one.
- What the template needs, from the last column above. Variables, datasources
  and content types are covered under [Set up what a template needs](#set-up-what-a-template-needs).

## Create, test and publish a template

These steps use `edge-geo-redirect`, which needs nothing, so it runs on any
install. Swap in the id of the template you chose.

1. List the templates with their tier, and whether this instance can save each:

   ```bash
   curl -s http://localhost:3001/api/admin/flows/templates \
     -H "Authorization: Bearer $TOKEN" \
     | jq -r '.data[] | "\(.id)\t\(.tier)\t\(.enabled)"'
   ```

   ```text
   join-three-tables-api	free	true
   nightly-sheet-export	pro	false
   ...
   edge-geo-redirect	free	true
   ```

2. Create a draft from the template. The create body takes only `name`, `slug`,
   `description` and `definition`, so pick those out of the list:

   ```bash
   curl -s http://localhost:3001/api/admin/flows/templates \
     -H "Authorization: Bearer $TOKEN" \
     | jq '.data[] | select(.id == "edge-geo-redirect")
           | {name: .name, slug: .definition.slug, description: .description, definition: .definition}' \
     | curl -X POST http://localhost:3001/api/admin/flows \
         -H "Authorization: Bearer $TOKEN" \
         -H "Content-Type: application/json" \
         -d @-
   ```

   The answer is `201` with the new flow as a `draft`. Copy its `id` into
   `FLOW_ID`. A slug the tenant already uses answers `409`, so change `slug` to
   take a second copy.

3. Test it with a sample request:

   ```bash
   curl -X POST http://localhost:3001/api/admin/flows/$FLOW_ID/test \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"trigger": {"headers": {"cf-ipcountry": "DE"}}}'
   ```

   The run succeeds, and the step `pick` has the output
   `{"to": "https://example.com/de"}`. A step that sends mail, writes or posts
   reports what it would do instead of doing it. A `GET` request runs for real.

4. Publish it:

   ```bash
   curl -X POST http://localhost:3001/api/admin/flows/$FLOW_ID/publish \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer is `200` with `"status": "active"`.

5. Call it. This template is public, so it answers on the public URL:

   ```bash
   curl -i "http://localhost:3002/api/v1/flows/p/$FLOW_ID" -H "CF-IPCountry: DE"
   ```

   ```http
   HTTP/1.1 302 Found
   Cache-Control: private, max-age=300
   Location: https://example.com/de
   ```

In the admin console, open **Flows**, choose **New flow**, give it a name and a
slug, and press **Create**. The editor opens empty. Choose **Use a template**
and press **Use** on a card. The flow keeps the name and slug you gave it, and
nothing is saved until you save.

## Set up what a template needs

Each template opens with a note on the canvas that lists its variables and
datasources. Create them before you test:

- **Variables:** `PUT /api/admin/flows/variables` with `key`, `value` and
  `is_secret`. Mark keys and tokens secret, so they are never shown again.
- **Datasources** (pro templates): `POST /api/admin/flows/datasources`, then
  `POST /api/admin/flows/datasources/{id}/test` to check the connection.
- **Content types:** the schema builder, the schema API, or a
  [schema preset](../reference/schema-presets.md) for `comments` and `forms`.

Only the `admin` and `super_admin` roles manage variables and datasources, from
a signed-in session. See [connect outside systems](../features/flows.md#connect-outside-systems).

## What to know about each template

### `join-three-tables-api`

Loads orders filtered by `?status=` (default `open`), then their customers and
shipments side by side, and joins them. Its slug is `orders-with-customers`.
The cache key is `?customer_id=`, which picks a cache entry but filters
nothing, and `?status=` is not part of it. Within 30 seconds a caller who
changes only `status` gets the earlier answer, so drop the `key` to cache per
URL. [Build a custom API with flows](build-a-custom-api-with-flows.md) builds
a smaller version step by step.

### `bulk-import`

Takes a body of at most 1 MiB and at most 500 rows. Anything but a non-empty
JSON array answers `400` with `body must be a non-empty JSON array of rows`.
The answer lists how many rows were imported and their ids.

- The rename step is the field mapping, target on the left, source column on
  the right:

  ```text
  { email: lower(trim(item['E-mail'])), name: trim(item['Name']), company: item['Company'] ?? '', plan: item['Plan'] ?? 'free' }
  ```

- Every row is a new entry. Nothing matches existing entries by email.
- Each row is validated against the content type. A row that fails stops the
  run, the step record names its index, and the rows before it stay written.
- The trigger is `auth: admin`, so call it at
  `/api/admin/flows/{id}/invoke` from a signed-in admin session.
- The new entries reach webhooks and search like any other write, but they do
  not start other flows.

### `scheduled-import`

Runs every hour on the hour, in UTC. It needs `flow-pro`, because the fetch
calls outside the instance.

- Set the variable `import_feed_url` to an address that answers `200` with a
  JSON array of at most 500 objects, and the secret variable
  `import_feed_token` to its bearer token. Any other answer fails the run with
  `the feed did not answer 200 with a JSON array of rows`.
- The rename step is the field mapping, target on the left, source column on
  the right:

  ```text
  { id: item['id'] ?? '', fields: { sku: trim(item['SKU']), name: trim(item['Name']), price: item['Price'] ?? 0 } }
  ```

- A row that carries the id of an entry updates that entry, and any other row
  creates one. Give the feed's rows their entry ids so an hourly run updates
  rather than duplicates.
- Set the upsert step's content type, `products` as shipped, to yours.
- Each row is validated against the content type. A row that fails stops the
  run, and the rows before it stay written.
- For a one-off file rather than a feed, use [bulk import](../features/bulk-import.md),
  which also reads CSV and Excel workbooks.

### `nightly-sheet-export`

Appends up to 1000 orders whose status is `paid` below the last row of the
`Orders` tab. The columns go in alphabetical order (`created_at`, `currency`,
`customer_id`, `id`, `total`), so lay out the header row to match. Each night
appends every paid order again, so filter on a date if you want only new ones.

### `enrich-on-create`

Runs after a `customers` entry is created, looks up its email domain, and
writes `company`, `industry`, `employees` and `enriched_at` back, so those
fields must exist. A customer without an email is logged and left alone.
Change the URL on the request node to your provider's. The lookup is a `GET`,
so a test run calls your provider for real.

### `notify-on-failure`

Mails the flow, the node that stopped it, the error and the run id. Set the
trigger's `schema` to one flow's slug to watch only that flow, or add a filter
such as `trigger.data.trigger_type == 'trigger.cron'`. A failure of this flow
is logged, never reported, so it cannot start itself.

### `cdn-purge`

Runs after an entry of any content type is updated, not on create or delete.
It purges two URLs: the entry's page (`/<type>/<slug or id>`) and its listing
(`/<type>`). The `cloudflare` datasource has the `base_url`
`https://api.cloudflare.com/client/v4` and bearer auth with an API token that
has the Zone.Cache Purge permission, or a global API key pair in its headers
as `X-Auth-Email` and `X-Auth-Key`. `site_url` is the scheme and host with no
trailing slash. The event carries the entry's own fields, not its publication
status, so to purge on a value of a `status` field of yours, add the filter
`trigger.data.status == 'published'`.

### `seo-sitemap` and `seo-robots`

Both are public `GET`s, limited to 60 calls a minute and cached for ten minutes.

- The sitemap lists up to 1000 `pages` entries whose `status` field is
  `published`, with `lastmod` from `updated_at`, and `changefreq` when a page
  has one. Slugs go into the XML as written, so keep them to letters, digits
  and hyphens.
- Leave `priority` out of the `pages` content type. The template compares it
  as a number, content nodes return it as a string, and a page with any
  `priority` value fails the run today.
- The robots file reads `robots_rules` (`user_agent`, `rule_type` of `allow`,
  `disallow` or `crawl-delay`, `path`, `sort_order`) and ends with a `Sitemap:`
  line. Line breaks are removed from values, so an entry cannot add a rule.

Both answer at `/api/v1/flows/p/{flow_id}`. Serve them at `/sitemap.xml` and
`/robots.txt` with a rewrite at your CDN or reverse proxy.

### `edge-geo-redirect`

Answers `302` to the page for the country in `cf-ipcountry`, which Cloudflare
sends, and to `https://example.com/` for every other country. Edit the country
map in the **Pick a destination** node, and change the header name for another
CDN.

### `review-workflow`

Runs when an editor saves a `reviews` entry with its `action` field set, moves
`stage`, clears `action`, and mails the reviewer or the author.

| Action | From | To | Mail to |
|---|---|---|---|
| `submit` | `draft` or `rejected` | `in_review` | reviewer |
| `approve` | `in_review` | `approved` | author |
| `reject` | `in_review` | `rejected` | author |
| `request_changes` | `in_review` | `draft` | author |

Any other move fails the run with `cannot <action> a review that is <stage>`.
The content type needs `title`, `body`, `author_email`, `reviewer_email`,
`stage` (`draft`, `in_review`, `approved`, `rejected`, default `draft`),
`action` and `comment`. For roles, publishing on approval and deadlines, use
[editorial review](../features/editorial-review.md).

### `comment-moderation`

Runs after a comment is created with no status or `pending`, posts it to the
`moderation` datasource's `/check`, and sets `status` to `spam` or `approved`
from `{"spam": true}` or `{"spam": false}`. An answer with no verdict leaves the
comment `pending` for a person. With [AI](../features/ai.md) licensed, swap the
request node for `ai.classify` with the labels `approved` and `spam`.

### `form-submitted`

Loads the submission's form, mails the submission when the form has a
`notify_email`, then posts it to the `form_webhook` datasource when the
variable `form_webhook` is `on`. The mail comes first, so a failed mail stops
the webhook too.

### CRM templates

Each runs after an entry is created. HubSpot matches contacts by email, so you
can copy that flow and set its event to `after_update` to keep edits in step.
Pipedrive and Salesforce create a new record on every run, so leave them on
create. HubSpot and Salesforce skip a customer without an email, and Salesforce
sends the email as the last name when a customer has no `last_name`.

### `chat-notify`

Posts the content type and event as the title, the entry's id and status as
fields, and the entry as JSON text, in each service's own format. Set
`chat_provider` and the `chat` datasource's `base_url`:

| Provider | `base_url` |
|---|---|
| `slack` | The incoming webhook URL |
| `discord` | The webhook URL |
| `telegram` | `https://api.telegram.org/bot<token>`, plus a `chat_id` on the datasource or the node |

Record values are written literally, so they cannot add formatting or mention a
group. To post to two services, copy the flow and set `channel` and
`datasource` on the copy's node. Set the trigger's schema to `*` to post every
content type.

### `cost-ledger`

Runs at 00:30 UTC. It reads the month's API request counts per key and
yesterday's AI calls from the `replica` datasource, and writes one `cost_entry`
per line. Point `replica` at a read replica or a nightly copy, because a
datasource on the instance's own database host is refused. The statements are
written for Postgres. Their `$1`, `$2` placeholders work on every database, but
the rest of the SQL may need small changes on MySQL or SQL Server. Each night
adds a new month-to-date API line, so sum only the latest one.

<details>
<summary>The cost_entry content type</summary>

| Field | `field_type` | Holds |
|---|---|---|
| `resource_type` | `text` | `api` or `ai` |
| `period_start` | `date` | First day of the period |
| `period_end` | `date` | The day the line was written |
| `amount` | `number` | The charge in `currency` |
| `currency` | `text` | `USD` |
| `unit_count` | `number` | Requests or tokens |
| `unit_label` | `text` | `requests` or `tokens` |
| `metadata` | `json` | The key and byte counts, or the model and call count |

</details>

## Save your own template

A flow you want to reuse becomes a template of the tenant. In the editor,
open **Use a template** and expand **Save a template**, or post it:

```bash
curl -X POST http://localhost:3001/api/admin/flows/templates \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/flows/orders.yaml"}'
```

Send either `url` or `document`, the definition as an object or as YAML or JSON
text, up to 1 MiB. The definition's slug names the template, and saving the
same slug again replaces it. Saved templates appear in the list with
`"source": "saved"`, and `DELETE /api/admin/flows/templates/{id}` removes one.

## Next

- [Build a custom API with flows](build-a-custom-api-with-flows.md): the join
  endpoint by hand, with caching and an API key.
- [Keep the content model and flows in git](schema-and-flows-as-code.md):
  export a flow made from a template and promote it.
- [Flow definition format](../reference/flow-definition.md): every node, its
  settings and its ports.
