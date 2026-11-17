---
title: "28. Back up, upgrade and scale"
description: "Back up the database and test a restore, export one tenant, upgrade the engine and the console in the right order, and add replicas when one is no longer enough."
sidebar:
  order: 28
covers:
  console: []
  api:
    - "POST /api/admin/tenant-backup/export"
    - "GET /readyz"
    - "GET /api/admin/pool/health"
verified:
  engine: v0.62.0
  console: v0.28.0
---

A running install needs three habits: a backup you have restored once, upgrades taken in the
right order, and a plan for when one replica is not enough. This lesson gives the order and
links the pages that hold each step. You need the deployment from
[lesson 27](configure-and-deploy.md) and an admin's token in `TOKEN` for the API steps.

## 1. Back up the database and the media

> **Included free** on every install.

Everything the engine stores is in its database, so back it up with your database's own tools
or your host's managed backups. Uploaded files are not in the database: back them up with it, or
keep them in a bucket with its own retention, as [Storage](../features/object-storage.md)
describes. Keep `ENCRYPTION_KEY` with the backup, because the stored credentials cannot be read
without it.

A backup is proven only by a restore. Restore one into a separate database, start an engine
against it, and read a few trips. The [production checklist](../deploy/production-checklist.md#database)
holds this item.

## 2. Export one tenant

> **Requires** a license with the `tenant-backup` feature. See [pricing](https://lyeve.com/pricing/).

A tenant export is one SQL document with the tenant's content types and entries, which you can
load into another tenant or another database:

```bash
curl -X POST http://localhost:3001/api/admin/tenant-backup/export \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"include_data": true, "target_dialect": "postgres"}' \
  -o tenant-backup.sql
```

[Back up and restore tenants](../how-to/backup-and-restore.md) goes on from here: check the
file, load it, run scheduled encrypted backups to a bucket, test restores and clone a tenant.

## 3. Upgrade the engine, then the console

> **Included free** on every install.

1. Read the upgrade section for the release you move to, in the guide you deployed with:
   [Docker images](../deploy/docker.md#upgrading-to-0620-and-console-0280),
   [Docker Compose](../deploy/docker-compose.md#upgrading-to-0620-and-console-0280) or
   [Kubernetes](../deploy/kubernetes.md#upgrading-to-0620-and-console-0280). Moving across
   several releases, read each section in turn.
2. Take a backup, as in step 1.
3. Upgrade the engine first. It applies its new migrations when it starts, and replicas that
   start together wait for each other.
4. Wait until the engine is ready, then upgrade the console:

   ```bash
   curl -s http://localhost:3001/readyz
   ```

5. Check the version the engine runs with `lyeve --version`, as
   [lesson 29](cli-and-sdk.md) shows.

To try a release before it is stable, run its release candidate on a copy of your data.
[Release candidates](../deploy/docker.md#release-candidates) says how they are tagged.

## 4. Add replicas when one is not enough

> **Included free** on every install. Listing the running replicas needs a license with the `cluster` feature. See [pricing](https://lyeve.com/pricing/).

Read [When one replica is enough](../scaling/index.md#when-one-replica-is-enough) first. Its
three numbers decide the next move. Pool waits are one of them:

```bash
curl http://localhost:3001/api/admin/pool/health \
  -H "Authorization: Bearer $TOKEN"
```

A `wait_count` that keeps growing means requests queue for a database connection, so
[size the pool](../scaling/index.md#size-the-connection-pool) before you add a replica. When
you do add one, [Run more than one replica](../scaling/index.md#run-more-than-one-replica)
lists what every replica shares: one signing key, one Redis and one bucket for media. Then
work down the rest of [Scale and tune](../scaling/index.md) in its order: read replicas,
caching, load shedding and rolling deploys.

## Next

[29. The CLI and the SDK](cli-and-sdk.md) reaches the engine from a shell and from code. For
depth, read [Back up and restore tenants](../how-to/backup-and-restore.md),
[Scale and tune](../scaling/index.md) and [Replication](../features/replication.md).
