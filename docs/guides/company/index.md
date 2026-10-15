---
title: Company and licensing guide
description: "Evaluate LyEve: find what is free, what a license adds, and the security and compliance pages a review needs, each by link."
sidebar:
  order: 1
---

This guide is for the people deciding whether to adopt LyEve: engineering leadership,
procurement and security review. LyEve is a headless CMS you host yourself, on your
infrastructure and your database, so your content, users and files stay in systems you control.

## Your first hour

1. [Core concepts](../../concepts/index.md): what LyEve does, in six ideas.
2. [Licensing and tiers](../../concepts/licensing-and-tiers.md): what runs free with no time
   limit, what each paid feature is called, and what happens when a license expires.
3. [Pricing](https://lyeve.com/pricing/): plans, and features you can buy on their own.
4. [Source code](../../trust/source-code.md): which parts are MIT, which are commercial, and
   which repositories you can read today.
5. [Security controls](../../trust/security-controls.md): each control as a route or setting a
   reviewer can check on a running install.
6. [Data protection](../../trust/data-protection.md): subject access and erasure, retention,
   legal holds, residency and masking.

## Common jobs

| I need to... | Read |
|---|---|
| Know what the free tier includes and where it stops | [Licensing and tiers](../../concepts/licensing-and-tiers.md) |
| Know what a license adds | [What a license adds](../../concepts/licensing-and-tiers.md#what-a-license-adds) |
| Confirm a signed license works with no network at all | [Verify offline](../../trust/verify-offline.md) |
| Answer a security questionnaire | [Security controls](../../trust/security-controls.md) |
| Answer a privacy review | [Data protection](../../trust/data-protection.md) |
| Keep data in one region | [Data residency](../../features/data-residency.md) |
| Prove who changed what | [Audit log](../../features/audit-log.md) |
| Connect our identity provider | [SAML single sign-on](../../features/saml-sso.md), [OAuth sign-in](../../features/oauth-sign-in.md), [SCIM provisioning](../../features/scim-provisioning.md) |
| Estimate what running it costs | [Free-tier overview](../../deploy/free-tier-overview.md) and [Deployment](../../deploy/index.md) |
| See what operating it involves | [Operator guide](../operator/index.md) |
| See what building on it involves | [Developer guide](../developer/index.md) |

## Three things to know

- **One image for every tier.** Every install runs the same engine image, and the license
  decides which paid features run and which free ceilings it lifts. Moving between tiers is a key change, not a migration.
- **The free tier is for production.** It has no time limit, and an expired license never takes
  a free feature away. A paid feature that stops keeps its data.
- **Nothing extra to run.** A license is verified on the instance. There is no license server to
  host, no sidecar and no agent. A license key, as opposed to a signed token, is exchanged for a
  token at start, and that exchange sends the key and a random install id, nothing else.

## Related

- [Features](../../features/index.md): every feature, and whether it is free.
- [lyeve.com/contact](https://lyeve.com/contact/): support and sales questions.
