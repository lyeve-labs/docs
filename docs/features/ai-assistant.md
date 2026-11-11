---
title: AI assistant
description: "Ask the assistant in LyEve Console about the page you have open, and accept or run the changes it prepares, within your own role."
sidebar:
  order: 2
capability_of: ai
---

> **Requires** a license with the `ai` feature. See [pricing](https://lyeve.com/pricing/).

The assistant is a panel in LyEve Console that answers questions about the page you have open
and about the engine, and prepares changes for you. It reads the engine through the engine's
own API, acting as you, so it sees only what your role may see. A change that publishes, deletes
or sends data out comes back to you as a request to review, and a value for a form goes into the
form for you to accept or reject. Every signed-in role can use it, a viewer included, each within
their own permissions.

:::caution[Beta]
AI is in beta, and the assistant with it.
:::

:::tip[Best for]
- **Finding how to do something.** Ask how to schedule an entry to publish later, and get an
  answer drawn from the operations your role may call.
- **Reading the state of your tenant.** Ask which flows failed today or which entries are still
  drafts. The assistant runs the reads itself.
- **Filling a form.** Ask for values for the entry you are writing. Each value goes into the form
  for you to accept or reject, following the field's help text.
- **Preparing a change for review.** Ask to publish three drafts, and choose **Run** on each
  request once you have read it.
:::

:::note[Use something else when]
- Your own code drafts, rewrites or translates entries. Call the content assistant route, as
  [AI](ai.md#use-ai-from-the-api) describes.
- You want a flow from a description. The flow editor has its own assistant, covered in
  [Build a flow with the assistant](../how-to/build-a-flow-with-the-assistant.md).
:::

## What it needs

| Needs | Who sets it | Where |
|---|---|---|
| A license with the `ai` feature | A super admin | See [licensing and tiers](../concepts/licensing-and-tiers.md) |
| An enabled AI provider | A super admin | **Operations > AI**, **Providers** tab. See [connect a provider](ai.md#connect-a-provider) |
| A monthly budget on that provider, for an editor, a viewer or a custom role | A super admin | The provider's `max_budget_usd`, above `0` |
| **AI for this tenant** and **Assistant** turned on | An administrator | **Operations > AI**, **Settings** tab. Both are on by default |

An admin and a super admin can ask the assistant without a budget on the provider. Anyone else
uses the provider the tenant's priority order picks, and only when that provider has a monthly
budget, because every question spends the tenant's money. Without one they read "The assistant
needs a monthly budget on the AI provider. Ask an administrator to set one."

## Try it

You need a license with `ai`, an enabled provider, and an account in LyEve Console.

1. Choose **Assistant** in the console's header. On a phone the header shows its icon alone.
   The panel opens beside the page.
2. Type a question under **Message**, such as `Which content types does this tenant have?`.
   Enter sends, and Shift+Enter starts a new line.
3. While it works, the panel names each step, such as "Look up what the engine can do", and how
   many are done. A task of several steps shows its plan under **To do**. **Stop** ends the
   answer, and anything the assistant had not finished is not done.
4. Read the answer. Under it the panel shows the model, the tokens used and the estimated cost.

The conversation stays while you move between pages. The **New conversation** button starts
another one.

## What it runs and what it asks first

| The operation | What the assistant does |
|---|---|
| A read, or a save to a draft that nobody sees until it is published, that your role may call | Runs it as you |
| An operation that changes data, deletes data or sends data out of the engine | Shows it as a **Request to review**. Nothing is sent until you choose **Run** |
| An operation only a super admin may call | Shows it as a request to review, for a super admin too |
| An operation your role may not call | Neither runs nor proposes it, and names the roles that may |

A request to review shows the method, the path, the query and the body it would send, and a badge
for its effect: **Changes nothing**, **Saves a draft**, **Changes data**, **Deletes data** or
**Sends data out of the engine**. **Run** sends it with your own session, and **Reject** sends
nothing.

One question may make at most 12 calls to the model and 24 steps. Past either, the assistant
stops and says so, and asking again continues.

## Fill a form

With a form open, such as a new entry, ask the assistant to fill it. The panel lists the values
under **Suggested values for** and the form's name, and puts each one into the form. Accept or
reject each value in the form, or choose **Accept all** or **Reject all**, then save there. The
assistant never saves the form for you.

A field's help text guides the value it suggests. In the schema builder, open a field's help
text and write up to 500 characters. Editors see it under the field, and the assistant follows
it.

## Keep a field from the assistant

In the schema builder, the **No AI** column marks a field the assistant never reads or fills.
Over the API, a field carries `"no_ai": true` in the content type's definition. The value of a
field marked **No AI** or **Private** never reaches the AI provider, whatever the assistant
reads, and the assistant cannot fill it. The same holds for a password or a file in a form, and
for any value whose key names a credential, such as a token or an API key. The model sees such a
value as `[redacted]`.

The flow node **Fill fields with AI** (`content.fill`) keeps to the same marks: it never asks
for a field marked **No AI** or **Private**.

## Turn the assistant off

An administrator turns the assistant off for one tenant while the rest of AI keeps working.

<div class="paths">
<div data-path="console">

Open **Operations > AI**, choose the **Settings** tab and turn off **Assistant**. It saves as
soon as you change it. Turning off **AI for this tenant** stops the assistant too.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/ai/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"assistant_enabled": false}'
```

A field you leave out keeps its value.

</div>
</div>

While it is off, the panel says "The assistant is off for this tenant." An administrator is
told to turn it on in AI settings, and anyone else to ask an administrator.

## In the builder

The [builder](builder.md) has an **Assistant** tab of its own. It reads the open site, its pages,
sections, theme and content types, and proposes changes to the open page's draft, which you
apply in the editor. It needs the same license, provider, budget and switches as the panel in
LyEve Console. Nothing that publishes, unpublishes, archives or deletes is offered to it. While a
content type marks a page's blocks **Private** or **No AI**, the assistant does not see them and
does not offer a draft save, because a save writes the whole page.

## Ask from the API

`POST /api/admin/ai/assistant` takes a signed-in session and nothing else. An API key is refused.
The [quickstart](../getting-started/quickstart.md) shows how to get a session token, which the
example keeps in `TOKEN`.

```bash
curl -N -X POST http://localhost:3001/api/admin/ai/assistant \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Which content types does this tenant have?", "mode": "drafts"}'
```

The answer is a stream of server-sent events:

```text
event: step
data: {"id":"call_1","title":"Look up what the engine can do","status":"running"}

event: message
data: {"text":"This tenant has three content types: post, author and page."}

event: done
data: {"conversation_id":"6f1c2a9e-3b7d-4c55-9a0e-1d2f3b4c5d6e"}
```

<details>
<summary>Request fields and events</summary>

| Field | Meaning |
|---|---|
| `prompt` | Required. The question, up to 32 KB |
| `conversation_id` | Continues a conversation you started from the same place. Empty in `done` while transcripts are off |
| `mode` | `ask`, the default: every change comes back as a proposal. `drafts`: an operation that is not a read or a draft is neither run nor proposed |
| `page` | `path` and `title` of the page the question is about |
| `form` | The open form: its `label` and `fields`, each with `name`, `label`, `help`, `kind`, `required`, `options` and `value`. A field with `withheld` set carries no value and is never filled |
| `provider_id` | The provider to ask, for an admin or a super admin. Ignored for anyone else |

The body is at most 1 MB, and a form at most 200 fields.

| Event | Carries |
|---|---|
| `message` | `text`, the assistant's words |
| `step` | `id`, `title`, `status` (`running`, `done` or `failed`) and, for an engine call, `operation_id` and `effect` |
| `plan` | `entries`, each with `status` (`now`, `next` or `done`), `title` and `detail` |
| `proposal` | A request to review, with `method`, `path`, `query`, `body` and `effect`. Form values come as `kind` `form` with `values` |
| `usage` | `model`, `tokens_in`, `tokens_out` and `cost` |
| `done` | `conversation_id` |
| `error` | `code` and `message`, and `retry_after` in seconds for `ai_rate_limited` |

An `error` event's `code` is one of `ai_budget_spent`, `ai_rate_limited`, `ai_off`,
`ai_no_provider`, `ai_unsupported`, `ai_not_licensed`, `conversation_gone` or `assistant_failed`.

</details>

<details>
<summary>Refusals before the stream opens</summary>

| Status | Message | When |
|---|---|---|
| `400` | `prompt is required`, `prompt is too long`, `mode must be ask or drafts`, `form has too many fields` | The request is malformed |
| `403` | `the assistant needs a signed-in session, not an API key` | The request carries an API key |
| `403` | `the assistant is switched off for this tenant` | **Assistant** is off |
| `403` | `an administrator has to set a monthly budget on the AI provider before you can use the assistant` | The asker is below admin and the provider has no monthly budget |
| `404` | `AI is switched off for this tenant` | **AI for this tenant** is off |
| `404` | `conversation not found` | `conversation_id` names a conversation you did not start here |
| `503` | `no enabled AI provider configured` | The asker is below admin and no provider is enabled |

</details>

Each conversation is kept as a transcript of kind `engine_assistant` while transcripts are on, as
[read and recap transcripts](ai.md#read-and-recap-transcripts) describes.

## Related

- [AI](ai.md): providers, budgets, prompts, transcripts and the tenant's switches.
- [Schema changes](schema-changes.md): where a content type's fields, help text and marks are
  set.
- [Builder](builder.md): the app whose **Assistant** tab works on a page's draft.
