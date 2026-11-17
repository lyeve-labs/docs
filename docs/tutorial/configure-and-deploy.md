---
title: "27. Configure and deploy the engine"
description: "Move from the laptop install to a server: pick a host and a database, set what the engine needs to start, run both images, check the engine answers, and choose which features start."
sidebar:
  order: 27
covers:
  console: []
  api:
    - "GET /readyz"
    - "GET /api/admin/setup"
    - "GET /api/admin/entitlements"
verified:
  engine: v0.62.0
  console: v0.28.0
---

> **Included free** on every install.

The console lessons ran on your computer. This lesson and the three after it are about the
engine on its own: where it runs, how it is set up, how it is kept safe and how you reach it
from code. Each step names the order to work in and the page that holds the detail, so keep
those pages open beside this one. You need a shell on the host you deploy to.
[Lesson 26](customize-the-console.md) is the one before this.

## 1. Pick a host and a database

Read [Choose where to run LyEve](../deploy/index.md). It compares the hosts, lists the
supported databases with their versions, and explains why the console needs a server of its own
rather than a static host. Settle three things before you go on:

- where the engine runs, and where the console runs beside it
- which database, PostgreSQL, MySQL or SQL Server, and who backs it up
- whether you start with one replica. [Scale and tune](../scaling/index.md#when-one-replica-is-enough)
  says when one is enough.

## 2. Write the settings the engine needs to start

Open [Configuration](../reference/configuration.md#start-here) and set its seven variables first.
Generate each secret once and store it where your deployment keeps secrets: a new
`ENCRYPTION_KEY` makes every stored credential unreadable.

Leave `APP_ENV` unset on a server. The engine then runs as production and refuses a weak
configuration, and [What production refuses](../reference/configuration.md#what-production-refuses)
lists each refusal with its fix. Read it now, because a refusal stops the start.

## 3. Run the engine and the console

Follow the guide for your host:

| You run on | Read |
|---|---|
| One server with Docker | [Docker Compose](../deploy/docker-compose.md), the shortest path to a working console |
| Containers you orchestrate yourself | [Docker images](../deploy/docker.md) |
| Kubernetes | [Kubernetes](../deploy/kubernetes.md) |
| A platform such as Fly.io, Render, Railway or Cloud Run | The engine guide for it under [Deployment](../deploy/index.md#compare-the-hosts) |
| No database, as a webhook relay | [Stateless mode](../deploy/stateless-mode.md) |

Each guide puts a TLS proxy in front, and each runs every pending migration when the engine
starts.

## 4. Check that the engine answers

From the host, ask the readiness probe and the setup route:

```bash
curl -s http://localhost:3001/readyz

curl -s -H 'X-Forwarded-Proto: https' http://localhost:3001/api/admin/setup
```

The probe answers `{"status":"ok", ...}` once the engine is ready. A new install answers the
setup route with `"setup_required": true`. Create the first administrator on the console's
setup page, as lesson 1 did, or with the setup token as the
[quickstart](../getting-started/quickstart.md) shows. [Verify](../deploy/docker.md#verify)
explains the header.

## 5. Choose which features start

A feature runs when the image includes it, the license carries it and `LYEVE_PLUGINS` asks for
it. Every feature is in the published image, so two settings decide:

- `LYEVE_LICENSE_KEY` holds your license, if you have one. Lesson 26 applied one from the
  console instead.
- `LYEVE_PLUGINS` narrows what starts, for a smaller deployment. Leave it empty to run every
  feature the license allows.

[Choose which features start](../concepts/licensing-and-tiers.md#choose-which-features-start)
explains both. Then check what the engine runs, with an admin's token in `TOKEN`:

```bash
curl http://localhost:3001/api/admin/entitlements \
  -H "Authorization: Bearer $TOKEN"
```

`state` is `free`, `active` or `grace`, and `features` lists every feature that runs.

## 6. Keep the configuration where you can review it

Environment variables suit secrets. Everything else can live in a YAML file that you keep in
version control, which the engine reads from `LYEVE_CONFIG` or its default places.
[The YAML file](../reference/configuration.md#the-yaml-file) shows the shape, and
[Change a setting](../how-to/change-a-setting.md) shows how a change applies in each deployment
and how to confirm the value the engine uses.

## Next

[28. Back up, upgrade and scale](backup-and-upgrade.md) keeps the install safe as it grows. For
depth, read the [Operator guide](../guides/operator/index.md) and
[Installation](../getting-started/installation.md).
