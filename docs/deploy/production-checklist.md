---
title: Production Checklist
description: "Tick off the secrets, proxies, database, backups and monitoring of a LyEve deployment before it takes real traffic."
sidebar:
  order: 13
---

Work through this before real users or real data reach a deployment. One line per item, each
linked to the page that explains it. With `APP_ENV` unset or `production`, the engine checks the
items marked "refused" itself and will not start until they pass.

## Secrets

- [ ] `JWT_SECRET` is a random value of at least 16 characters (refused). [Configuration](../reference/configuration.md#start-here)
- [ ] `ENCRYPTION_KEY` is a different random value of at least 32 characters, stored durably (refused). [Configuration](../reference/configuration.md#start-here)
- [ ] `LYEVE_AUDIT_HMAC_KEY` is exactly 64 hex characters (refused). [Configuration](../reference/configuration.md#start-here)
- [ ] `ADMIN_CONSOLE_KEY` holds the same value on the engine and the admin console. [Configuration](../reference/configuration.md#sign-in-tokens-and-passwords)
- [ ] `LYEVE_SETUP_TOKEN` is set when more than one replica can answer first-run setup. [Configuration](../reference/configuration.md#sign-in-tokens-and-passwords)
- [ ] Secrets live in a secret store or an uncommitted `.env` file. [Secure your instance](../how-to/secure-your-instance.md)

## Tokens and cookies

- [ ] The signing key at `JWT_KEY_PATH` is on a volume, or the same file on every replica. [Kubernetes](kubernetes.md#the-signing-key)
- [ ] The signing key file is readable by its owner only (refused). [Configuration](../reference/configuration.md#what-production-refuses)
- [ ] `JWT_EXPIRY_SECS` is a deliberate choice of `3600` or less (refused above). [Secure your instance](../how-to/secure-your-instance.md)
- [ ] `SECURE_COOKIE=true`, and the deployment is served over HTTPS end to end (refused). [Docker images](docker.md#reverse-proxy-and-tls)
- [ ] `PASSWORD_HASH_ALGO` and the password rules are chosen on purpose. [Secure your instance](../how-to/secure-your-instance.md)

## Network and proxies

- [ ] `LYEVE_CONSOLE_URL` is the admin console's public `https` URL. [Configuration](../reference/configuration.md#network-and-proxies)
- [ ] `LYEVE_BASE_URL` is the engine's public URL. [Configuration](../reference/configuration.md#network-and-proxies)
- [ ] `CORE_INTERNAL_URL` and `CORE_API_INTERNAL_URL` are set on the admin console. [Docker images](docker.md#the-admin-image)
- [ ] `CORS_ORIGINS` lists the exact origins of your browser apps (`*` refused). [Secure your instance](../how-to/secure-your-instance.md)
- [ ] `TRUSTED_PROXIES` names your proxy or load balancer CIDRs. [Secure your instance](../how-to/secure-your-instance.md)
- [ ] `RATE_LIMIT_RPS` is set (refused unset). [Secure your instance](../how-to/secure-your-instance.md)
- [ ] Your proxy sets `X-Forwarded-Proto: https`, on internal plain-HTTP hops too. [Docker images](docker.md#reverse-proxy-and-tls)
- [ ] The Admin API (port 3001) has no public host unless a tool outside your network needs it. [Docker images](docker.md#reverse-proxy-and-tls)
- [ ] `super_admin` is held by as few accounts as possible. [Secure your instance](../how-to/secure-your-instance.md)

## Database

- [ ] The database user is not `postgres`, `root` or `sa` (refused). [Supported databases](index.md#supported-databases)
- [ ] `DATABASE_URL` uses `sslmode=require` or stronger outside a private network. [Configuration](../reference/configuration.md#what-production-refuses)
- [ ] `DATABASE_MAX_CONNECTIONS` times the replicas stays under the database's limit. [Configuration](../reference/configuration.md#database)
- [ ] The database is backed up, and one restore has been tested. [Backup and restore](../how-to/backup-and-restore.md)
- [ ] Media is backed up with the database, or stored in a bucket with its own retention. [Object storage](../features/object-storage.md)

## More than one replica

- [ ] Every replica uses the same signing key file. [Kubernetes](kubernetes.md#the-signing-key)
- [ ] `CACHE_DRIVER=redis` and `REDIS_URL` are set, and the boot log's `refresh token store` line says sessions are shared. [Scaling](../scaling/index.md)
- [ ] `RATE_LIMIT_BACKEND=redis` is set. [Rate limiting](../features/rate-limiting.md)
- [ ] Media goes to an S3-compatible bucket. [Object storage](../features/object-storage.md)

## Email

- [ ] Outbound email is configured if users rely on invitations, password resets or notifications. [Email](../features/email.md)

## Features and license

- [ ] `LYEVE_LICENSE_KEY` is set if you use paid features. [Licensing and tiers](../concepts/licensing-and-tiers.md)
- [ ] `LYEVE_PLUGINS` is empty or names exactly the features you run. [Licensing and tiers](../concepts/licensing-and-tiers.md)
- [ ] **Plugins** in the admin console, or `GET /api/admin/plugins/status`, shows the features you expect. [Licensing and tiers](../concepts/licensing-and-tiers.md)

## Monitoring

- [ ] Readiness probes and uptime checks use `/readyz`, liveness probes `/healthz`. [Health endpoints](docker.md#health-endpoints)
- [ ] Metrics are scraped from `GET /api/admin/metrics` with `METRICS_TOKEN`. [Configuration](../reference/configuration.md#observability)
- [ ] Logs go somewhere durable, not only to container output. [Logs](../features/logs.md)

## Free hosting

- [ ] If any part runs on a free plan, its limits are acceptable. [Free-tier stack](free-tier-overview.md#limits-of-the-free-stack)
