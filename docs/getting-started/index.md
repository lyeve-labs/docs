---
title: Getting Started
description: "Learn what LyEve is, then follow the getting started pages in order from installing it to modeling your first content."
sidebar:
  order: 1
---

LyEve is a headless CMS. It stores your content in your own database and serves it as JSON
over HTTP. It has no public website of its own: you build the site, app or service that shows
the content, and that client reads it from LyEve. Anything that can make an HTTP request can
use it, from a site built with a JavaScript framework or a static site generator to a mobile
app or another backend.

You define your own content types while the engine runs, with no code change and no deploy.
The free tier is a complete CMS for production use, with no time limit. A license key adds
the paid features and lifts the limits of the free ones. [Licensing and tiers](../concepts/licensing-and-tiers.md) lists both
sides.

## The path

Read these in order. Each page starts where the one before it stops.

| Step | Page | What it gives you |
|---|---|---|
| 1 | This page | What LyEve is. The [three pieces you run](../index.md#what-you-run) are on the front page |
| 2 | [Installation](installation.md) | The engine running against your database, the admin console beside it, and the settings production requires |
| 3 | [Quickstart](quickstart.md) | Your first administrator, a token, a content type and an entry, in about ten minutes |
| 4 | [Your first content type](first-content-type.md) | A small blog: posts linked to authors, a change previewed and applied, a post published and read back with its author |
| 5 | [Run the engine and the console together](engine-and-console.md) | The engine, the admin console and a database on your computer, a tour of the console's screens, and how to change a setting |

:::tip[Only trying it out?]
The Quickstart starts its own database and engine on Docker, so you can begin there and come
back to Installation when you set up a real instance.

Prefer screens to `curl`? [Run the engine and the console together](engine-and-console.md)
starts the console as well and does every step in the browser.
:::

## After the path

- [Core concepts](../concepts/index.md): the ideas the rest of the docs assume.
- [Features](../features/index.md): everything LyEve can do, what each feature is best for,
  and whether it needs a license.
- [Choose your path](../index.md#choose-your-path): the guide for the job you are doing, as
  a developer, an operator, an editor or an evaluator.
