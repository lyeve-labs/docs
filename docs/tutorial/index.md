---
title: Tutorial
description: "Learn LyEve from the first start to production in thirty lessons, building a small travel guide on the engine and LyEve Console as you go."
sidebar:
  order: 0
---

This tutorial walks every screen of LyEve Console and the engine routes behind it, in the
order you meet them. You build one small project as you go: a travel guide with trips and the
stops along each one. Each lesson starts where the last one ended, so read them in order the
first time.

Every task shows two tabs. **Console** takes the steps in LyEve Console, and **Engine API**
takes the same steps with `curl`. Pick the one you want, and the choice follows you from page
to page. Where a screen moves, a short clip shows it.

Most of LyEve is free. A lesson says at the top of each section whether it needs a license,
and where a screen looks different without one, it shows both. [Licensing and
tiers](../concepts/licensing-and-tiers.md) explains how a license is applied.

## Start

| Lesson | What you do |
|---|---|
| [1. Run the engine and the console](run-lyeve.md) | Start LyEve on your computer, create the first super admin and sign in |
| [2. Find your way and secure your account](find-your-way.md) | Read the dashboard, add a second factor, trust a device and recover a password |

## Content

| Lesson | What you do |
|---|---|
| [3. Model your content](model-content.md) | Create the `trip` and `trip_stop` content types and link them |
| [4. Write and publish](write-and-publish.md) | Write trips and stops, publish them and restore an earlier version |
| [5. Media](media.md) | Upload photos, describe them, publish them and choose where files are stored |
| [6. Reviews and releases](reviews-and-releases.md) | Send a trip through review and publish several entries at once |
| [7. Search](search.md) | Search the guide, add synonyms and let visitors search it |
| [8. Languages and activity](languages-and-activity.md) | Translate a trip and see who changed what |
| [9. Import, export and config sync](import-and-export.md) | Load stops from a file, export the guide and move its setup to another engine |

## Delivery

| Lesson | What you do |
|---|---|
| [10. API access and keys](api-access.md) | Give a site an API key, give a script an admin token and browse the API reference |
| [11. GraphQL, gRPC, streams and realtime](graphql-and-streams.md) | Query trips over GraphQL and gRPC and follow changes as they happen |
| [12. Webhooks, events and email](webhooks-and-email.md) | Call your own endpoint on a change, replay events and send email |
| [13. Flows](flows.md) | Build a custom endpoint and an automation on the flow canvas |
| [14. Experiments, analytics and recommendations](experiments-and-analytics.md) | Test two versions of a page, read API analytics and recommend trips |
| [15. Connected editors](connected-editors.md) | Connect LyEve Builder to the instance |

## Access

| Lesson | What you do |
|---|---|
| [16. Users, roles and permissions](users-and-roles.md) | Invite an editor and decide what each role may do |
| [17. Sign-in, SSO and provisioning](sign-in-and-sso.md) | Add OAuth and SAML sign-in, SCIM provisioning, a captcha and the security settings |
| [18. Privacy, compliance and the audit log](privacy-and-audit.md) | Answer a privacy request, mask personal data and read the audit log |

## Operations

| Lesson | What you do |
|---|---|
| [19. Jobs and features](jobs-and-features.md) | Run scheduled jobs and see which features run |
| [20. Rate limits](rate-limits.md) | Set request limits and read who was refused |
| [21. AI](ai.md) | Connect a model provider, tune the prompts and read the transcripts |
| [22. Tenants and costs](tenants-and-costs.md) | Add a tenant, set its quotas and follow what it costs |
| [23. Logs, errors and alerts](logs-and-errors.md) | Read the log, triage errors and get alerted |
| [24. Performance](performance.md) | Find slow queries, profile a request and tune the cache |
| [25. Captures, synthetic checks and telemetry](captures-and-telemetry.md) | Capture and replay requests, probe the instance and export metrics |
| [26. Customize the console](customize-the-console.md) | Apply a license, change settings and shape the console for your team |

## The engine

| Lesson | What you do |
|---|---|
| [27. Configure and deploy the engine](configure-and-deploy.md) | Set the engine's configuration and deploy it to a server |
| [28. Back up, upgrade and scale](backup-and-upgrade.md) | Back up, move to a new release and run more than one replica |
| [29. The CLI and the SDK](cli-and-sdk.md) | Call the engine from the command line and from TypeScript |
| [30. Run it in production](production.md) | Check an instance before real traffic reaches it |

## Next

Start with [1. Run the engine and the console](run-lyeve.md). To look up a feature rather than
learn the product in order, go to [Features](../features/index.md).
