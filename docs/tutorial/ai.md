---
title: "21. AI"
description: "Turn AI on for your tenant, connect a model provider with your own key, set the prompt a kind of call runs under, price the model, make a call about a trip and read its transcript."
sidebar:
  order: 21
covers:
  console:
    - "/admin/ai/settings"
    - "/admin/ai/providers"
    - "/admin/ai/providers/[id]"
    - "/admin/ai/prompts"
    - "/admin/ai/prompts/[use_case]"
    - "/admin/ai/prices"
    - "/admin/ai/transcripts"
    - "/admin/ai/transcripts/[id]"
  api:
    - "GET /api/admin/ai/settings"
    - "PUT /api/admin/ai/settings"
    - "POST /api/admin/ai/providers"
    - "GET /api/admin/ai/providers/{id}/models"
    - "GET /api/admin/ai/providers/{id}/capabilities"
    - "POST /api/admin/ai/prompts/{use_case}"
    - "PUT /api/admin/ai/prices/{kind}/{model}"
    - "POST /api/admin/ai/summarize"
    - "GET /api/admin/ai/transcripts"
    - "GET /api/admin/ai/transcripts/{id}"
    - "POST /api/admin/ai/transcripts/{id}/recap"
verified:
  engine: v0.62.0
  console: v0.28.0
---

> **Requires** a license with the `ai` feature. See [pricing](https://lyeve.com/pricing/).

In this lesson you connect a model provider with your own key, decide what a summary of a trip
should sound like, tell the engine what the model costs, and read the record of a call. You
need a super admin account, a key from OpenAI or another provider, and a token in `TOKEN` for
the API steps. [Lesson 20](rate-limits.md) is the one before this.

Nothing is sent to a provider until you add one. Every call then goes to the tenant's enabled
providers, highest priority first.

On an install without the `ai` feature, AI is missing from the sidebar, and each AI address
shows **AI is not included in this build** instead of its screen.

![The AI settings address on a free install, which says AI is not included in this build.](./media/ai-settings-free.png)

![AI settings on a licensed install, with the three switches for this tenant and the retention window.](./media/ai-settings-licensed.png)

## Check that AI is on for this tenant

<div class="paths">
<div data-path="console">

1. Open **Operations > AI > AI settings**.
2. Under **This tenant**, check that **AI for this tenant** is on. **Store transcripts** keeps
   the text of each call, and **Assistant** lets everyone who signs in ask the assistant.

Each switch saves as soon as you change it. Under **Retention**, a super admin sees how long
transcripts are kept and where that value comes from.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/ai/settings \
  -H "Authorization: Bearer $TOKEN"
```

To turn a switch, send only the field you change:

```bash
curl -X PUT http://localhost:3001/api/admin/ai/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"transcripts_enabled": true}'
```

</div>
</div>

With **AI for this tenant** off, every other AI screen says **AI is switched off for this
tenant** and offers the same switch to turn it back on.

## Add a provider

A super admin adds providers. The key is encrypted and never shown again.

<div class="paths">
<div data-path="console">

1. Open **Operations > AI > Providers** and choose **New provider**.
2. Enter `openai` in **Name** and choose **OpenAI** under **Kind**.
3. Paste your key in **API key** and enter `gpt-4o-mini` in **Default model**.
4. Enter `10` in **Priority**, turn **Enabled** on and choose **Create**.

The provider's own page opens, with its kind, its priority and a **Stored** key.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/ai/providers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "openai", "kind": "openai", "api_key": "sk-your-key", "default_model": "gpt-4o-mini", "enabled": true, "priority": 10}'
```

The answer is `201` with the provider and `"has_key": true`. Copy its `id` into `PROVIDER_ID`.
`enabled` is `false` unless you send it.

</div>
</div>

![Opening New provider, choosing OpenAI, pasting a key, entering the default model and priority, and creating the provider.](./media/ai-new-provider.webm)

The form shows the fields each kind needs:

| Field | What it does |
|---|---|
| **Kind** | OpenAI, Anthropic, or a server that speaks one of their APIs |
| **Preset** | For an OpenAI-compatible server, fills **Base URL** and **Key header**, such as Ollama or OpenRouter |
| **Base URL**, **Key header**, **Query string** | Where a compatible server listens, and how it takes the key |
| **Default model** | Used when a request does not name a model |
| **Priority** | When a request does not name a provider, higher priority is tried first |
| **Modalities** | What this provider may be used for: text, embeddings or images |
| **Monthly budget** | Dollars per billing period. Requests over it are refused with a `402` |
| **Rate limit** | Requests per minute. `0` means no limit |
| **Allow a private address** | Lets the base URL resolve to a private or loopback address, for a server on your network |

[Connect a provider](../features/ai.md#connect-a-provider) lists what each kind can do.

## See what the provider offers

<div class="paths">
<div data-path="console">

1. On **Providers**, choose **Manage** on the `openai` row.
2. Under **Models**, choose **List models**. The list is read from the provider and not saved.
3. Under **Capabilities**, enter `gpt-4o-mini` in **Model** and choose **Check**.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/ai/providers/$PROVIDER_ID/models \
  -H "Authorization: Bearer $TOKEN"

curl "http://localhost:3001/api/admin/ai/providers/$PROVIDER_ID/capabilities?model=gpt-4o-mini" \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

**Configuration** on the same page edits the provider, and **Delete** removes it.

![A provider's page, with its configuration, the models it lists and what one model can do.](./media/ai-provider.png)

## Set the prompt for summaries

Each kind of call runs under the prompt of its use case. The shipped text is version 0, and each
save is a new version that applies from then on.

<div class="paths">
<div data-path="console">

1. Open **Operations > AI > Prompts** and choose **Manage** on the `summarize` row.
2. Replace the text in **System prompt** with
   `Summarize a trip for a travel newsletter in two sentences. Name the places.`
3. Choose **Save as version 1**. The badge reads **Using version 1**.

**Versions** lists every saved text. The button on an older row makes that version current
again, as a new version. The guardrail under **Guardrail, always appended** is added to every
prompt and cannot be removed.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/ai/prompts/summarize \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"body": "Summarize a trip for a travel newsletter in two sentences. Name the places."}'
```

The answer is `201` with the new `version`, which is now active.

</div>
</div>

![Opening the summarize prompt, editing the system prompt, saving it as version 1 and seeing it in the versions list.](./media/ai-prompt-version.webm)

## Price the model

The engine estimates the cost of each call from a price table. A model with no price is
estimated at zero, so add a row for each model you use. A super admin sets the prices, and they
apply to the whole engine.

<div class="paths">
<div data-path="console">

1. Open **Operations > AI > Prices** and choose **New price**.
2. Choose **OpenAI** under **Kind** and enter `gpt-4o-mini` in **Model**.
3. Enter your vendor's current prices in **Input per 1K** and **Output per 1K**, in dollars per
   thousand tokens, and leave **Image per call** at `0`.
4. Choose **Create**.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/ai/prices/openai/gpt-4o-mini \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input_per_1k": 0.00015, "output_per_1k": 0.0006, "image_per_call": 0}'
```

</div>
</div>

![The Prices page with one row for gpt-4o-mini and its input and output prices.](./media/ai-prices.png)

## Summarize a trip

The summarize route has no control in the console, so this step is an API call. Paste the
description of one of your trips as `content`:

```bash
curl -X POST http://localhost:3001/api/admin/ai/summarize \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "Four days in Lisbon: the Alfama on foot, a tram ride to Belem, a day trip to Sintra and dinner in Bairro Alto.", "style": "concise"}'
```

The answer holds the `summary`, the `model`, the token counts, a `cost_estimate` from the price
table and a `transcript_id`. The text depends on the provider and the model.

## Read and recap the transcript

<div class="paths">
<div data-path="console">

1. Open **Operations > AI > Transcripts**. The newest call is first. **Kind**, **Subject kind**
   and **Subject id** narrow the list, and **Filter** applies them.
2. Choose **Open** on the newest row. The page shows the messages, the tokens in and out, and
   the cost estimated from the price table.
3. Choose **Recap**. The same provider writes a short recap, which is stored with the
   transcript.

**JSON** and **Markdown** download the transcript.

</div>
<div data-path="api">

```bash
curl "http://localhost:3001/api/admin/ai/transcripts?kind=route&limit=1" \
  -H "Authorization: Bearer $TOKEN"
```

Copy the `id` into `TRANSCRIPT_ID`, then read it and ask for the recap:

```bash
curl http://localhost:3001/api/admin/ai/transcripts/$TRANSCRIPT_ID \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/ai/transcripts/$TRANSCRIPT_ID/recap \
  -H "Authorization: Bearer $TOKEN"
```

</div>
</div>

![Opening the newest transcript, reading its messages and totals, and choosing Recap.](./media/ai-transcript-recap.webm)

With **Store transcripts** off, the list says so and links to the settings, and calls return no
`transcript_id`. Cost and latency are still recorded.

## Next

[22. Tenants and costs](tenants-and-costs.md) gives a partner agency a tenant of its own. For
depth, read [AI](../features/ai.md), [AI assistant](../features/ai-assistant.md) and
[Build a flow with the assistant](../how-to/build-a-flow-with-the-assistant.md).
