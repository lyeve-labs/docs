---
title: LyEve documentation
description: "Learn what LyEve Core and LyEve Console each do, try them in ten minutes, and find where your own work starts."
sidebar:
  order: 1
---

LyEve is a headless CMS you run yourself. You define the shape of your content, write it in
the browser or over the API, and your sites, apps and services read it as JSON. It runs on
your own servers against your own database, so the content and the accounts stay with you.
The core CMS and a set of features are free with no time limit. A license key adds the paid
features, and the same images serve both.

## The products

| Product | What it is | Where it comes from |
|---|---|---|
| [LyEve Core](concepts/index.md) | The engine. It stores your content, accounts and tenants on PostgreSQL, MySQL or SQL Server and serves them over an API. MIT licensed. | The image `ghcr.io/lyeve-labs/lyeve-core` |
| [LyEve Console](console/index.md) | The browser app your team works in: write and publish content, manage users and roles, and configure the engine. | The image `ghcr.io/lyeve-labs/lyeve-admin` |

The engine has no screens of its own. It answers two HTTP APIs: the Admin API manages the
instance, and the Content API reads and writes content. The console calls the Admin API for
you, so everything it does is also open to a script. The database is yours to provide:
PostgreSQL, MySQL 8.0 or 8.4, or SQL Server and Azure SQL, on your own server or a managed
service. [Core concepts](concepts/index.md#the-two-apis) explains the two APIs, and
[Source code](trust/source-code.md) says what is published and under which license.

## Where you start

| You are... | Start here | Then |
|---|---|---|
| Trying it for the first time | [Quickstart](getting-started/quickstart.md) | [Getting started](getting-started/index.md) puts the first pages in order |
| Writing or publishing content | [LyEve Console](console/index.md) | [User guide](guides/user/index.md) |
| Building a site or app on it | [Developer guide](guides/developer/index.md) | [Your first content type](getting-started/first-content-type.md) and the [SDK](sdk/index.md) |
| Running it in production | [Operator guide](guides/operator/index.md) | [Deployment](deploy/index.md) and the [production checklist](deploy/production-checklist.md) |
| Evaluating it for your company | [Company and licensing guide](guides/company/index.md) | [Licensing and tiers](concepts/licensing-and-tiers.md) |

## How these docs are arranged

- **Start**: this page, [Getting started](getting-started/index.md) and the
  [Company and licensing guide](guides/company/index.md).
- **Features** and **How-to guides**: each task once, with the steps in the console and the
  steps with the API side by side. [Features](features/index.md) lists every feature and
  whether it is free.
- **LyEve Console**: the [overview](console/index.md) and the
  [User guide](guides/user/index.md) for the people who work in the browser.
- **LyEve Core**: [concepts](concepts/index.md), the
  [Developer](guides/developer/index.md) and [Operator](guides/operator/index.md) guides,
  the [SDK](sdk/index.md), [integrations](integrations/index.md),
  [deployment](deploy/index.md), [scaling](scaling/index.md), the reference pages and
  [trust](trust/security-controls.md).

## Find something specific

- **[Configuration](reference/configuration.md)** and **[API endpoints](reference/api-endpoints.md)**:
  every setting and every route.
- **[Glossary](reference/glossary.md)**: short definitions of the terms these pages use.
- **[Pricing](https://lyeve.com/pricing/)**: plans and prices.
