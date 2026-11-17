---
title: "30. Run it in production"
description: "Take a deployment to real traffic in order: work through the checklist, harden the instance, point the probes, confirm the security controls are enforcing, and watch it."
sidebar:
  order: 30
covers:
  console: []
  api:
    - "GET /healthz"
    - "GET /readyz"
    - "GET /api/admin/security/controls"
    - "GET /api/admin/metrics"
verified:
  engine: v0.62.0
  console: v0.28.0
---

> **Included free** on every install. The web application firewall and trusted devices need a license with the `waf` and `device-fingerprint` features. See [pricing](https://lyeve.com/pricing/).

This last lesson takes the deployment from [lesson 29](cli-and-sdk.md) to real users and real
data. Work through the steps in order, because each one assumes the one before it. You need a
super admin's token in `TOKEN` for the API steps.

## 1. Work through the production checklist

Open the [production checklist](../deploy/production-checklist.md) and tick each line. The
lines marked refused are the ones the engine checks itself, so a deployment that starts has
passed them. The rest are yours: the console's address, the proxy, the database user, backups,
email, and more than one replica if you run them.

## 2. Harden the instance

Follow [Harden your instance](../how-to/secure-your-instance.md) from its first step to its last.
It sets the secrets, token signing and lifetimes, TLS and cookies, CORS, the host header and
trusted proxies, rate limits, API keys, the password policy, and who may read the audit, metrics
and debug routes. Each step ends with a check that the setting took effect.

## 3. Point the probes at the health endpoints

Orchestrators and uptime monitors use the two probes the engine serves without a token:

```bash
curl -s http://localhost:3001/healthz
curl -s http://localhost:3001/readyz
```

Point liveness at `/healthz` and readiness at `/readyz`. A route under `/api` needs a token, so a
prober pointed at one marks a healthy engine down.
[Health endpoints](../deploy/docker.md#health-endpoints) says when each answers `503`. Add a
[synthetic probe](captures-and-telemetry.md#check-the-site-on-a-schedule) to watch the public
site from outside.

## 4. Confirm the security controls are enforcing

```bash
curl http://localhost:3001/api/admin/security/controls \
  -H "Authorization: Bearer $TOKEN"
```

Each control's `status` is `PASS` when it is enforcing. Any other status carries `remedy`, the
sentence that turns it on. **Settings > Security** in the console shows the same report, as
lesson 17 did. [Security controls](../trust/security-controls.md#is-it-enforcing) explains
every state.

## 5. Watch it

Decide where each signal goes before the first incident:

- **Metrics.** Scrape `GET /api/admin/metrics` with `METRICS_TOKEN`, or push them to a
  collector, as [Metrics export](../features/metrics-export.md) shows.
- **Logs.** Ship them somewhere durable, as [Logs](../features/logs.md#send-logs-to-loki-or-elasticsearch)
  shows. [Lesson 23](logs-and-errors.md) set up the volume and spike alerts.
- **Errors and slowness.** [Lesson 23](logs-and-errors.md) and [lesson 24](performance.md)
  walked through the screens.

[Watch a running instance](../guides/operator/index.md#watch-a-running-instance) maps each
question to the tool that answers it.

## 6. Know where to look when something breaks

Keep [Troubleshooting](../guides/developer/troubleshooting.md) at hand. It reads an error body,
follows a request ID through the log, and lists the errors people meet most. Keep the backup
and the restore you tested in [lesson 28](backup-and-upgrade.md) at hand too.

## Next

You have finished the tutorial. Go back to [the tutorial](index.md) to revisit any lesson, or
open the [Operator guide](../guides/operator/index.md) for every operating job in one place.
