---
title: LyEve documentation
description: "Learn what LyEve is and who it is for, try it in ten minutes, and find the guide for the job you are doing."
sidebar:
  order: 1
---

LyEve is a headless CMS you run yourself. You define the shape of your content, write it in
the admin console or over the API, and your sites, apps and services read it as JSON. It
runs on your own servers against your own database, so the content and the accounts stay
with you.

It is for developers who build sites and apps on content, operators who run the instance,
editors who write in the admin console, and teams that want one content store behind several
brands or customers. The core CMS and a set of features are free with no time limit. A
license key adds the paid features, and the same images serve both.

## Choose your path

| I want to... | Start here |
|---|---|
| Try it in ten minutes | [Quickstart](getting-started/quickstart.md) |
| Model my content | [Your first content type](getting-started/first-content-type.md), then [the data model](concepts/data-model.md) |
| Build a site or app on it | [Developer guide](guides/developer/index.md) |
| Automate work with flows | [Flows](features/flows.md) and the [flow templates](how-to/flow-templates.md) |
| Run it in production | [Operator guide](guides/operator/index.md) and the [production checklist](deploy/production-checklist.md) |
| Edit content as an editor | [User guide](guides/user/index.md) |
| Evaluate it for my company | [Company guide](guides/company/index.md) and [licensing and tiers](concepts/licensing-and-tiers.md) |
| See everything it can do | [Features](features/index.md) |

New to LyEve? [Getting started](getting-started/index.md) puts the first pages in order, and
[core concepts](concepts/index.md) explains the handful of ideas the rest of the docs assume.

## What you run

| Piece | What it does | Where it comes from |
|---|---|---|
| Engine | Stores your content and accounts in your database, and answers every request over two HTTP APIs: one to manage the instance, one to read and write content. It has no screens of its own | The image `ghcr.io/lyeve-labs/lyeve-core` |
| Admin console | The web interface where editors write and administrators configure. It calls the engine for you | The image `ghcr.io/lyeve-labs/lyeve-admin` |
| Database | Holds everything: PostgreSQL, MySQL 8.0 or 8.4, or SQL Server and Azure SQL | You provide it, on your own server or a managed service |

Your own code can call the APIs directly or through the TypeScript packages on npm
([SDK overview](sdk/index.md)). [Core concepts](concepts/index.md#the-two-apis) explains the
two APIs, and [Source code](trust/source-code.md) says what is published and under which
license.

## Find something specific

- **[How-to guides](how-to/index.md)**: task walkthroughs such as adding a webhook or backing
  up an instance.
- **[Deployment](deploy/index.md)**: Docker Compose, Kubernetes and hosted platforms.
- **[Configuration](reference/configuration.md)** and **[API endpoints](reference/api-endpoints.md)**:
  every setting and every route.
- **[Glossary](reference/glossary.md)**: short definitions of the terms these pages use.
- **[Pricing](https://lyeve.com/pricing/)**: plans and prices.
