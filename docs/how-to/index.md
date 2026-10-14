---
title: How-to guides
description: "Pick a step-by-step recipe for webhooks, admin tokens, flows, content migration, backups and hardening, and see what you end up with."
sidebar:
  order: 1
---

Each recipe starts from one goal and walks you to a working result, following one reasonable
path rather than every option. They assume LyEve is running. If it is not, start with
[Installation](../getting-started/installation.md) and the
[Quickstart](../getting-started/quickstart.md). To learn what a feature does, read its page in
[Features](../features/index.md). To look up a route or a setting, use the reference.

## The recipes

Each recipe stands on its own. Do them in any order.

| Recipe | What you end up with | Needs |
|---|---|---|
| [Send webhooks on events](add-a-webhook.md) | An endpoint of yours called when content changes, with the signature checked and failed deliveries recovered | Free. Retry policies and resending need `webhook-pro`, and replaying a window needs `events` |
| [Change a setting](change-a-setting.md) | A setting changed where your deployment keeps it, applied with the right restart, and confirmed in the console | Free |
| [Automate the Admin API with admin tokens](admin-tokens.md) | A credential for a CI job or script: chosen grants, one tenant, an expiry of at most 90 days, rotation and a request log | Free |
| [Back up and restore tenants](backup-and-restore.md) | One tenant exported, validated and restored, backup jobs writing encrypted archives to S3, and a tenant cloned | `tenant-backup` |
| [Build a custom API with flows](build-a-custom-api-with-flows.md) | One `GET` endpoint joining three content types, tested, cached, rate limited, published and moved to another tenant | Free. Custom paths and outbound nodes need `flow-pro` |
| [Build a flow with the assistant](build-a-flow-with-the-assistant.md) | A flow you described in plain words, drafted, checked, accepted onto the canvas and refined | `ai` |
| [Start from a flow template](flow-templates.md) | One of the eighteen starter flows created as a draft, supplied with what it needs, and published | Free. Templates that call outside need `flow-pro` |
| [Migrate existing content](migrate-existing-content.md) | Content from WordPress, Strapi, Contentful, a file or another LyEve instance in your content types, or the whole database on another server | Free. Moving the whole database needs `migration-toolkit` |
| [Keep the content model and flows in git](schema-and-flows-as-code.md) | The model and the flows as files in your repository, a CI check that stops a change that drops data, and promotion to production | Free |
| [Harden your instance](secure-your-instance.md) | The secrets, signing, cookies, CORS, proxy, rate limit, API key and password settings a public instance needs | Free. The firewall and trusted devices need `waf` and `device-fingerprint`, and passkey enrollment needs `mfa-pro` |

## Conventions

- **Ports.** Commands use the default ports: `3001` for the Admin API under `/api/admin/` and
  `3002` for the Content API under `/api/v1/`. [Core concepts](../concepts/index.md) explains
  the two.
- **Tokens.** Commands read a token from a shell variable such as `$TOKEN`. The
  [quickstart](../getting-started/quickstart.md) shows how to get one, and an
  [admin token](admin-tokens.md) suits a script.
- **Placeholders.** Values in angle brackets, such as `<tenant-slug>`, stand for your own.
  Replace them before you run a command.
- **Databases.** Every recipe works on PostgreSQL, MySQL 8.0 or 8.4, and SQL Server or Azure SQL.

To check that an instance answers on both ports before you start:

```bash
# Content API: the public key set, no credential needed
curl -s http://localhost:3002/.well-known/jwks.json

# Admin API: the content types, with a token
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3001/api/admin/schemas
```

## Related

- [Features](../features/index.md): what each feature does and whether it is free.
- [Developer guide](../guides/developer/index.md): the path for code that calls LyEve.
- [Deployment](../deploy/index.md): Docker, Docker Compose, Kubernetes and the hosted platforms.
