---
title: Replication
description: Keep the caches of several replicas in step through the shared database, and see which replicas are running.
sidebar:
  order: 8
---

> **Requires** a license with the `cluster` feature. See [pricing](https://lyeve.com/pricing/).
>
> Beta. Keeping caches in step and the list of running replicas work. Known gaps:
>
> - The list of running replicas is read through the API only. The admin console has no screen
>   for it.
> - Replication carries cache flushes and control notices only. Sessions, sign-in lockouts, rate
>   limits and realtime events are shared between replicas only through Redis. See
>   [Scaling](../scaling/index.md).

Each replica keeps permissions, content lists and license state in its own
memory. Without replication, two replicas on one database drift apart until
each cache expires: a permission rule saved on one is served stale by the
other. With replication on, a change made on one replica reaches the others in
about a second.

:::tip[Best for]
- **Changing a permission and having every replica enforce it at once.**
  Permission cache flushes reach every replica.
- **Publishing content behind a load balancer.** Content cache flushes reach
  every replica, so no replica serves the old version until its cache expires.
- **Seeing which replicas are alive.** The roster lists each running replica
  with its last heartbeat.
:::

:::note[Use something else when]
- You run one replica. It does not need replication.
- You are planning a deployment of several replicas. Replication is one part
  of it, and [scale and tune](../scaling/index.md) lists everything replicas
  share, Redis included.
:::

## How it works

Replicas exchange short control messages through the database they already
share, a handful a minute and never user traffic:

- permission cache flushes
- content cache flushes
- license renewals
- tenant feature changes
- notices that other features send to their own copies on other replicas

Nothing else is needed: no Redis and no message broker, on PostgreSQL, MySQL
and SQL Server alike. Each replica looks for new messages every second. A
replica that was briefly unreachable catches up on what it missed when it
returns, as long as it is back within ten minutes. Older messages are deleted.

## Try it

This needs a license with `cluster` on every replica, and a super admin token
in `TOKEN`. The [quickstart](../getting-started/quickstart.md) shows how to get
one.

1. Start every replica against the same database. If you set
   `LYEVE_PLUGINS`, include `cluster` on each one. See
   [licensing and tiers](../concepts/licensing-and-tiers.md).

2. List the running replicas:

   ```bash
   curl http://localhost:3001/api/admin/cluster/instances \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   {
     "count": 2,
     "instances": [
       {
         "instance_id": "lyeve-7f4c9b2-1",
         "hostname": "lyeve-7f4c9b2",
         "started_at": "2026-09-28T04:11:02Z",
         "last_heartbeat": "2026-09-28T04:19:57Z",
         "version": "v0.50.4",
         "listen_addr": ":3002",
         "status": "running"
       }
     ]
   }
   ```

3. Stop one replica cleanly and list again. It leaves the list at once.

Each replica sends a heartbeat every five seconds, and a replica is listed
while its last heartbeat is under fifteen seconds old.

## Settings

| Variable | What it does | Default |
|---|---|---|
| `INSTANCE_ID` | The name this replica shows in the roster and stamps on the messages it sends. Set it for stable names, such as the pod name. | hostname and process id |

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| A replica is missing from the list | It has not sent a heartbeat for fifteen seconds, or it runs without `cluster`. Check its logs and its `LYEVE_PLUGINS`. |
| A replica fails to start with a message about the database | Replication needs the shared database. Check that replica's `DATABASE_URL`. |
| `503` with `could not read the cluster roster` | The database did not answer. Retry. |
| `403` on the instances route | The caller is not a super admin. |

## Related

- [Scale and tune](../scaling/index.md): everything more than one replica
  needs.
- [Caching](caching.md): the caches replication keeps in step.
- [Access rules](access-rules.md): the permission rules every replica
  enforces.
