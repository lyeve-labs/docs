---
title: Build a flow with the assistant
description: "Describe a flow in plain words in the flow editor, read the validated draft, accept it onto the canvas, refine it, and ask about any node."
sidebar:
  order: 11
---

> **Requires** a license with the `ai` feature. See [pricing](https://lyeve.com/pricing).

Instead of dragging nodes onto the canvas, you can tell the flow editor what
you want and get a draft back. This guide builds one flow that way: when a
support ticket is created, classify it and email the right team. The assistant
drafts against your instance's own node catalog and checks every draft before
you see it. It never saves or publishes anything: you accept, save, test and
publish, as with a flow you drew by hand.

## Before you start

- **AI is set up.** Your license carries `ai`, the tenant's AI switch is on
  under **Operations > AI > AI settings**, and the tenant has an enabled
  provider. See [AI](../features/ai.md#connect-a-provider).
- **The nodes you ask for are licensed.** The example uses `ai.classify` and
  `email.send`, which need `flow-pro` as well. The assistant may draft a node
  your license does not cover, and then saving the flow answers `402`. See
  [free and flow-pro](../features/flows.md#free-and-flow-pro).
- **An email sender** is configured, for the example's mail. See
  [email](../features/email.md).
- **A `tickets` content type** with `subject` and `body` fields.

## 1. Open the Assistant tab

Open **Flows** in the admin console and open a flow, new or existing. The
drawer below the canvas has the tabs **Test**, **Runs**, **Assistant** and
**Guide**. **Assistant** opens with a box headed "Describe the flow" and a note
saying what is sent: the flow on the canvas, the node catalog and your prompt,
to the provider configured for this tenant.

What the tab says after you press **Send** tells you what is missing:

| The tab says | What to fix |
|---|---|
| "The assistant is switched off" | The tenant's AI switch is off under **Operations > AI > AI settings**, or the license does not carry `ai`. |
| "The assistant is not available on this engine" | Read the message under it. `no enabled AI provider configured` means the tenant needs an enabled provider. |
| `request blocked by WAF` | See the caution below. |

:::caution[Web application firewall]
With the [web application firewall](../features/web-application-firewall.md)
licensed, its injection rules refuse the assistant's request with `403`,
because the node catalog in the body looks like an attack. Read the rule ids
on `/api/admin/ai/assist/flow` in the violation log, and
[add an exception](../features/web-application-firewall.md#add-an-exception)
for each one on the path `/api/admin/ai/assist/flow*`.
:::

## 2. Describe the flow

Say what starts it, what it reads and what it does. Name your content types,
fields and addresses, because the assistant knows the catalog, not your content
model:

```text
When a tickets record is created, classify its body as billing, support or
sales, then email billing@example.com for billing and support@example.com for
everything else, with the ticket's subject in the email subject.
```

Press **Send**. The panel shows "Drafting the flow" while the provider answers,
then a line with the model, the tokens and the estimated cost of the turn.

## 3. Read the draft

The answer is a read-only YAML draft beside your prompt. Yours will differ in
names and layout, but it looks like this:

```yaml
version: 1
name: Route new tickets
slug: route-new-tickets
trigger:
  type: trigger.event
  config:
    kind: content
    schema: tickets
    event: after_create
nodes:
  - id: classify
    type: ai.classify
    position: { x: 120, y: 200 }
    config:
      input: "{{ trigger.data.body }}"
      labels: [billing, support, sales]
  - id: route
    type: control.switch
    position: { x: 400, y: 200 }
    config:
      cases:
        - port: billing
          when: input.label == 'billing'
  - id: email_billing
    type: email.send
    position: { x: 680, y: 120 }
    config:
      to: [billing@example.com]
      subject: "Billing ticket: {{ trigger.data.subject }}"
      text: "{{ trigger.data.body }}"
  - id: email_support
    type: email.send
    position: { x: 680, y: 280 }
    config:
      to: [support@example.com]
      subject: "Ticket: {{ trigger.data.subject }}"
      text: "{{ trigger.data.body }}"
edges:
  - { from: classify, to: route }
  - { from: route, from_port: billing, to: email_billing }
  - { from: route, from_port: default, to: email_support }
```

Under the draft, the panel either says "The validator accepts this draft." or
lists what is still wrong. Each line names the node, or `flow`, with the path
to the setting, then the problem:

```text
email_billing/config/to   required
```

Clicking a line brings that part of the draft into view, and selects the node
on the canvas when one with that id is there. The assistant gets one repair
pass: the problems go back to the model once, and whatever remains is shown
rather than hidden. **Accept** stays disabled while a problem remains, and says
why.

## 4. Accept it

Press **Accept**. The draft replaces what is on the canvas, and the editor says
"Draft accepted. Save to keep it." The flow keeps its own slug. Nothing is
stored until you save, and leaving the page without saving drops the draft.
**Discard** closes the draft and the conversation instead.

Then follow the usual path: save, run a test on the **Test** tab with a sample
trigger payload, and publish. In a test run `ai.classify` answers the first
label and calls no provider, unless you turn on live side effects.

## 5. Refine it

After the first answer the box reads "Refine the draft". Say what to change
rather than describing the flow again:

```text
Also send sales tickets to sales@example.com, and put the label in every
email subject.
```

Each answer is checked the same way and replaces the draft in the panel. The
canvas changes only when you accept. **Start over** clears the conversation.
Refining needs the tenant to store transcripts, because the conversation is
kept as one. With storage off, each prompt starts fresh from the canvas.

## 6. Ask about a node or a problem

- **A node.** Select it on the canvas and press the wand button in the
  inspector, "Ask the assistant about this node". The answer explains what the
  node does in this flow, what it receives and what it produces.
- **A problem.** Open the problem count in the toolbar and press "Ask the
  assistant about this problem" beside a line. The answer explains it and how
  to fix it.

Answers appear as text in the **Assistant** tab. The assistant never changes
the canvas for you.

## What is recorded, and what is sent

Every draft, follow-up and question is part of a transcript of kind
`flow_assistant`, with the prompt, the answer, the model, the tokens, the
latency and the estimated cost of each turn. Read them under
**Operations > AI > Transcripts**, or with
`GET /api/admin/ai/transcripts?kind=flow_assistant`. They follow the retention,
switches and privacy handling on the [AI](../features/ai.md#read-and-recap-transcripts)
page.

The request to your provider carries the tenant's `flow_assistant` prompt, the
node catalog, the definition on the canvas, your prompt, the earlier turns of
the conversation and any problems found in the draft. Content entries,
provider keys and other tenants' flows never go. See
[what never leaves your instance](../features/ai.md#what-never-leaves-your-instance).

From a script, the same calls are `POST /api/admin/ai/assist/flow` for a draft
and `POST /api/admin/ai/assist/flow/explain` for a question. Both need the
`admin` role and the catalog in the body as `catalog`, the text
`GET /api/admin/flows/catalog/llm` returns.

## Next

- [The flow catalog for language models](../reference/flow-catalog-for-language-models.md):
  draft flows with a model of your own instead.
- [Flow definition format](../reference/flow-definition.md): every node and
  setting the assistant can use.
- [Flow templates](flow-templates.md): starter flows, among them
  `comment-moderation`, which can use `ai.classify`.
