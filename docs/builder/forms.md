---
title: Take form posts
description: "Put a form on a builder page that saves each post in a content type or starts a flow, and see what publish refuses and what visitors see."
sidebar:
  order: 5
---

A form block on a LyEve Builder page posts from the visitor's browser straight to your engine.
Each post goes to the one place the form names: an entry of a content type, or a run of a flow.
An administrator chooses the content types and flows forms may use, and publish refuses a form
that sends anywhere else.

## Where a form can send

| Destination | What each post does |
|---|---|
| **Form submissions** | Adds an entry to Form submissions, the content type of the forms preset. The post's fields are kept together in its `data` field, so the form keeps whatever fields it has |
| Another content type | Adds an entry to that content type. Each form field fills the column of its name |
| A flow | Starts the flow with each field's value in its trigger. No content type stores the post |

The builder lists Form submissions first once it is allowed. [The forms preset](../reference/schema-presets.md#the-forms-preset)
describes its two content types, Forms and Form submissions.

A content type that anyone can read through public delivery takes no form posts, so it is never
offered as a destination.

## Allow destinations

An admin or a super admin chooses where forms may send.

<div class="paths">
<div data-path="console">

1. Open **Settings**, choose **Builder connections**, then the **Builder settings** tab.
2. Under **Where forms can send**, choose **Set up form submissions**. It adds the Forms and
   Form submissions content types and lets forms send to Form submissions. A tenant that already
   has Form submissions is only allowed to send to it.
3. Under **Content types**, tick the other content types forms may save posts into. Under
   **Flows**, tick the flows forms may start.
4. Choose **Save**.

**Flows** lists only the flows whose trigger is the builder form event. A warning beside a
ticked flow says when it no longer exists, starts from another trigger or is not active,
because a post to it then starts nothing.

</div>
<div data-path="api">

Apply the [forms preset](../reference/schema-presets.md) first if forms should send to Form
submissions. Then name the destinations in `form_schemas` and `form_flows` on the builder's
settings. The route takes a session token, as [Connect the builder](connect.md) describes, and
a field you leave out keeps its value:

```bash
curl -X PUT http://localhost:3001/api/admin/builder/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"form_schemas": ["form_submissions", "quote_requests"], "form_flows": ["new-lead"]}'
```

Each list holds at most 50 names, and an empty list lets forms send nowhere. A content type
served on public delivery is refused with "Forms cannot write into quote_requests, because the
public delivery routes serve its entries." A flow is checked only for the shape of its slug, so
make sure each slug names an active flow whose trigger is the builder form event.

</div>
</div>

Taking a content type or a flow off the list stops its forms at once. A visitor's post to one
of them then answers a plain not-found page.

## Build a form

In the builder, add a **Form** block to a page, then set it up in **Properties**:

1. Under **Send posts to**, choose **Content type** or **Flow**, then the one this form sends to.
2. Under **Fields**, keep 1 to 12 fields. Each has a **Name**, a **Label**, a **Type** and
   **Required**.
3. Set the **Button label** and the **Message after sending**.

A field's **Name** starts with a lower-case letter and holds up to 40 lower-case letters,
digits and underscores. With a content type it is the column the value fills, and the builder
builds each field from a column you pick, so its name and type fit from the start. With Form
submissions or a flow it is the key the value is kept under.

A form with no destination publishes, but takes no posts. The builder warns "This form has no
destination yet. Choose a content type or a flow for its posts."

## Field types

| Type | What the visitor enters | Longest value | Columns it fills |
|---|---|---|---|
| **Text** | One line of text | 200 characters | `text` |
| **Email** | One email address | 254 characters | `email`, `text` |
| **Phone** | A phone number | 40 characters | `text` |
| **Long text** | Several lines of text | 5,000 characters | `text` |
| **Checkbox** | A box to tick | | `boolean`, `text` |
| **Number** | A plain number, such as `-12` or `0.5`, with no exponent. With **Whole numbers only** on, a number with no decimal point | 40 characters | `number`, `decimal`, `text`, and `integer` with **Whole numbers only** on |
| **Date** | A calendar day | | `date`, `text` |
| **Link** | A full `http` or `https` address | 2,048 characters | `url`, `text` |
| **Choice** | One of its options, from a list that opens on "Choose one" | | `text`, or a column whose allowed values hold every option |

A **Choice** has 1 to 50 options. Each has a **Value** of up to 200 characters, which is what a
post stores and differs from every other option's, and a **Label** of up to 80 characters,
which is what the visitor reads.

When you pick a column, an `integer` column gives a **Number** with **Whole numbers only** on,
a `number` or `decimal` column a **Number**, a `date` column a **Date** and a `url` column a
**Link**. The builder offers no field for a `datetime`, `uid`, `media`, `json`, `relation` or
`rich_text` column.

### A choice and a column's allowed values

A column with an `enum` rule (see [Fields](../concepts/data-model.md#fields)) takes only a
**Choice**, and every option's value has to be one of the rule's values. The column may be
`text`, `email`, `url`, `date`, `integer`, `number` or `decimal`. Values are compared as text,
so an `integer` column that allows 1 to 5 takes the options `1` to `5`. A `boolean` or `json`
column with an `enum` rule takes no form field.

When you pick such a column, the builder gives it a **Choice** with one option for each allowed
value, labeled with the value. You can change the labels and remove options, and an option can
only be one of the column's values.

## What publish refuses

[Publish](publishing.md) checks every form on the pages and sections it publishes. A form that
cannot post refuses the whole action with `422`, and nothing is published. The answer names
each refused form under `blocks`, with its block `id`, the `field` it concerns and a `code`. The
builder shows each refusal beside the form's destination or the field it names.

| Code | Why |
|---|---|
| `form_destination_invalid` | The destination is not exactly one content type or one flow |
| `form_destination_not_listed` | The content type or flow is not on the administrator's lists, or the content type was deleted or opened to public delivery since |
| `form_preset_missing` | The form sends to Form submissions, but the tenant does not hold the whole forms preset. Apply it in the schema builder, then publish |
| `form_field_not_column` | A field's name is not a column of the content type |
| `form_field_type` | The column cannot store the field's type, or the column has allowed values and the field is not a **Choice** |
| `required_column_missing` | A required column with no default has no required field. A **Checkbox** always sends a value, so it fills a required column whether or not it is required |
| `form_field_options` | A **Choice** does not have 1 to 50 options with a value of up to 200 characters and a label of up to 80 |
| `form_option_not_allowed` | An option's value is not one of its column's allowed values |

Publish runs these checks again on every publish, so a form that published once is refused when
its content type changes under it.

## What a visitor sees

A post that is stored brings the visitor back to the form, which shows the **Message after
sending**. The page moves focus to that message, so a screen reader reads it out. The page needs
no script for any of it.

The visitor's browser checks each field's length and format, and whether it is required, before
it sends. Your engine checks them again, along with the content type's own rules, such as a
`min`, a `max` or a pattern. When it refuses a post, the form comes back with every value the
visitor typed, each refused field marked with what to fix, and one of these lines:

| Why | What the visitor reads |
|---|---|
| A field's value | Your message was not sent. Correct the marked fields and send it again. |
| The page was open for more than an hour | Your message was not sent because the page was open too long. Send it again. |
| The form was already sent from this page | Your message was already sent from this page. Send it again if you meant to. |
| Too many posts | Your message was not sent because too many messages arrived at once. Wait a few minutes, then send it again. |

The typed values come back once, within 10 minutes. A reload or a later return draws the form
empty and asks for the message again.

One address may post 5 times a minute and 30 times an hour. A post past that comes back with the
form empty, because nothing of it is kept. A site takes 1,000 posts a day and a tenant 5,000,
and a post past either comes back with its values.

:::note[Testing a form]
A post sent less than three seconds after the page was drawn is treated as automated. The
visitor sees the message after sending, and nothing is stored.
:::

## Manage posts

Posts are managed on LyEve Console's **Content** screens. Open **Content** and pick the
content type the form sends to.

A post to Form submissions is an entry whose `form` is the form it came from, whose `data` holds
the fields and whose `status` starts as `new`. Publish keeps one entry in **Forms** for each form,
named after its page's title or its section's name, with the form's purpose or block id added
where two forms on one page need telling apart. On that entry you set `notify_email`, which the `form-submitted` flow
template mails each new post to, and `active`. A post to a form whose entry is not active
answers a plain not-found page. The [forms preset](../reference/schema-presets.md#the-forms-preset)
lists every field.

An erasure or an export for a person reaches their posts: in Form submissions through the email
addresses in `data`, and in another content type through the form's first **Email** field.

## Start a flow

A flow that forms can start has an event trigger on the system event `builder.form.submitted`.
It starts only for a form whose destination names its slug, and only in the tenant the form was
posted to. The trigger's `data` carries `site`, `path`, `block`, `flow`, `submitted_at` and
`data`, which holds each field's value as text under its name, so a node reads an email field as
`{{ trigger.data.data.email }}`. Starting a flow from a form is free, while mail and chat nodes
need the `flow-pro` feature. See [Flows](../features/flows.md).

## What your site needs

A form takes posts only on a page your site draws when it is asked for, from a server that holds
the site's form signing key. In a site made from the builder's Astro starter:

1. Run `pnpm form-key`. It prints a `LYEVE_FORM_SIGNING_KEY=` line and the public key that goes
   with it.
2. Set that line where the site's server runs and where it builds. Never give it to a static
   build or commit it.
3. Set `LYEVE_PAGES` to `request`, so each page is drawn when it is asked for.
4. Set `LYEVE_ENGINE_URL` to your engine's address as visitors' browsers reach it, on the
   tenant's own hostname. Forms post to that address, and the engine finds the tenant from it.
5. Give the key in `LYEVE_API_KEY` the scope `builder.form_refusals:read` beside its read
   scopes, so the server can read a refused post back and fill the form again.
6. Build and deploy. The build tells the engine that the site takes form posts and sends the
   public half of the key. From then on, the builder offers the **Form** block.

The engine takes a post only from the site's **Production address** or **Allowed origin**, as
set in the builder's **Site settings**. To replace the key, run `pnpm form-key` again and
redeploy. Forms drawn before that deploy stop taking posts.

A page built ahead of time draws its forms disabled, and so does the builder's preview, which
says "Messages are sent from the published site, not from this preview." A published page whose
server holds no key says "This form is not accepting messages yet." A page with a form that can
post is sent with `Cache-Control: private, no-store`, because each drawing carries a form token
of its own that the engine takes once.

## Related

- [Show content entries](content.md): list entries and give each entry a page, where a form can
  sit too.
- [Publish](publishing.md): the other checks a publish runs.
- [Schema presets](../reference/schema-presets.md): the forms preset and the `form-submitted`
  flow template.
- [Flows](../features/flows.md): build the flow a form starts.
