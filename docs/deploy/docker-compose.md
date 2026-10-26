---
title: Docker Compose
description: "Run the engine, the admin console, Postgres and a TLS proxy on one host with one Compose file."
sidebar:
  order: 14
---

This page gives you the whole stack in two files: the engine
(`ghcr.io/lyeve-labs/lyeve-core`), the admin console (`ghcr.io/lyeve-labs/lyeve-admin`), a
Postgres database and a Caddy proxy that obtains TLS certificates. Copy both files, write the
`.env`, and run `docker compose up -d`.

:::tip[Best for]
- **Production on one server**, with the admin console and TLS included.
- **Evaluating everything** on a machine you control, the console's screens included.
:::

:::note[Use something else when]
You need more than one engine replica or failover. Use [Kubernetes](kubernetes.md), and read
[Scaling](../scaling/index.md).
:::

## Before you start

- A server with Docker and Docker Compose, with ports 80 and 443 open.
- Two DNS names pointing at it, such as `admin.yourdomain.com` and `api.yourdomain.com`.

## The Compose file

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    restart: unless-stopped
    # The database serves TLS so the engine connects with sslmode=require.
    # The Debian-based image ships a self-signed certificate, which
    # sslmode=require accepts.
    command:
      - -c
      - ssl=on
      - -c
      - ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
      - -c
      - ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
    environment:
      POSTGRES_USER: lyeve
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: lyeve
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U lyeve -d lyeve"]
      interval: 10s
      timeout: 5s
      retries: 5

  engine:
    image: ghcr.io/lyeve-labs/lyeve-core:latest
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://lyeve:${DB_PASSWORD}@db:5432/lyeve?sslmode=require
      JWT_SECRET: ${JWT_SECRET}
      ENCRYPTION_KEY: ${ENCRYPTION_KEY}
      LYEVE_AUDIT_HMAC_KEY: ${AUDIT_HMAC_KEY}
      ADMIN_CONSOLE_KEY: ${ADMIN_CONSOLE_KEY}
      RATE_LIMIT_RPS: "100"
      SECURE_COOKIE: "true"
      CORS_ORIGINS: https://admin.yourdomain.com
      LYEVE_CONSOLE_URL: https://admin.yourdomain.com
      TRUSTED_PROXIES: 172.28.0.0/24
      LYEVE_LICENSE_KEY: ${LYEVE_LICENSE_KEY:-}
      LYEVE_PLUGINS: ${LYEVE_PLUGINS:-}
    volumes:
      - engine-state:/var/lib/lyeve
      - uploads-data:/app/uploads
    depends_on:
      db:
        condition: service_healthy

  admin:
    image: ghcr.io/lyeve-labs/lyeve-admin:latest
    restart: unless-stopped
    environment:
      ORIGIN: https://admin.yourdomain.com
      CORE_INTERNAL_URL: http://proxy:8080
      CORE_API_INTERNAL_URL: http://proxy:8080
      ADMIN_CONSOLE_KEY: ${ADMIN_CONSOLE_KEY}
      ADDRESS_HEADER: X-Forwarded-For
      XFF_DEPTH: "1"
    depends_on:
      - proxy

  proxy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
    depends_on:
      - engine

networks:
  default:
    ipam:
      config:
        - subnet: 172.28.0.0/24

volumes:
  db-data:
  engine-state:
  uploads-data:
  caddy-data:
```

The proxy reads a `Caddyfile` next to the Compose file:

```text
# Caddyfile

# Reachable inside the Compose network only. The console's server sends its
# engine calls here. The hop is plain HTTP, so the proxy tells the engine the
# browser's side was HTTPS. Without that, SECURE_COOKIE=true answers every
# call with a redirect.
http://proxy:8080 {
  handle /api/admin/* {
    reverse_proxy engine:3001 {
      header_up X-Forwarded-Proto https
    }
  }
  handle {
    reverse_proxy engine:3002 {
      header_up X-Forwarded-Proto https
    }
  }
}

# The admin console. The browser also calls the engine's APIs on this origin.
admin.yourdomain.com {
  handle /api/admin/* {
    reverse_proxy engine:3001
  }
  handle /api/* {
    reverse_proxy engine:3002
  }
  handle /.well-known/* {
    reverse_proxy engine:3002
  }
  handle {
    reverse_proxy admin:3002
  }
}

# The Content API for your sites and apps.
api.yourdomain.com {
  reverse_proxy engine:3002
}
```

Replace `yourdomain.com` in both files with your domain, so Caddy can obtain certificates for
both public names. `TRUSTED_PROXIES` names the subnet the Compose file gives its network. If you
change one, change the other.

## Secrets and the `.env` file

Compose fills `${VAR}` from a `.env` file next to the Compose file. Keep every secret there and
keep that file out of version control. Generate it once:

```bash
cat > .env <<SECRETS
DB_PASSWORD=$(openssl rand -hex 24)
JWT_SECRET=$(openssl rand -hex 32)
ENCRYPTION_KEY=$(openssl rand -hex 32)
AUDIT_HMAC_KEY=$(openssl rand -hex 32)
ADMIN_CONSOLE_KEY=$(openssl rand -hex 32)
LYEVE_LICENSE_KEY=
LYEVE_PLUGINS=
SECRETS
```

Leave `LYEVE_LICENSE_KEY` empty to run the free features only. An empty `LYEVE_PLUGINS` starts
every feature the license allows. Store `JWT_SECRET` and `ENCRYPTION_KEY` somewhere durable:
`ENCRYPTION_KEY` encrypts stored secrets such as MFA seeds and OAuth client secrets, so a new
value makes them unreadable.

Each engine and console variable in these files is explained on
[Docker images](docker.md), and every engine variable on
[Configuration](../reference/configuration.md). `APP_ENV` defaults to `production`, which
refuses a weak configuration, as
[Configuration](../reference/configuration.md#what-production-refuses) lists.

## Persistence

Four volumes hold state:

- `db-data` holds the database. Every tenant, entry and user lives here.
- `engine-state` holds the signing key and the cached license. Lose it and every issued token
  stops working, so every user signs in again. Keeping it does not keep sessions through a
  restart: refresh tokens live in the engine's memory unless `CACHE_DRIVER=redis` and
  `REDIS_URL` point at a Redis, so after a restart everyone signs in again once their access
  token expires, 15 minutes by default.
- `uploads-data` holds uploaded media when storage is on local disk, the default.
- `caddy-data` holds the proxy's TLS certificates.

Back up `db-data` and `uploads-data` together, and keep a copy of `engine-state`. To keep media
off the host, point storage at an S3-compatible bucket instead. See
[Object storage](../features/object-storage.md).

## Bring it up

```bash
docker compose up -d
docker compose logs -f engine       # watch migrations and boot
docker compose ps                   # engine shows (healthy)
```

The engine waits for the database health check, because it applies its migrations when it
starts. Its image carries its own health check, which asks `/healthz`. The image has no shell,
so do not replace that check with a `CMD-SHELL` one.

The engine maps no host port, so ask for readiness from inside the network. The Caddy image has
`wget`:

```bash
docker compose exec proxy wget -qO- http://engine:3001/readyz
```

The answer is `{"status":"ok", ...}` with its checks once the engine is ready.
[Health endpoints](docker.md#health-endpoints) says what each probe checks.

## Create the first administrator

Creating the first administrator takes a setup token. The engine prints it at startup while no
account exists:

```bash
docker compose logs engine | grep setup_token
```

To choose the token yourself, set `LYEVE_SETUP_TOKEN` (16 characters or more) on the engine.

<div class="paths">
<div data-path="console">

Open `https://admin.yourdomain.com`. A fresh instance sends you to `/setup`. Paste the token
into **Setup token**, enter your email address, and the password in **Password** and again in
**Confirm password**, then choose **Create super admin account**. The console signs you in.

</div>
<div data-path="api">

```bash
curl -X POST https://admin.yourdomain.com/api/admin/setup \
  -H "Content-Type: application/json" \
  -d '{"email": "you@yourdomain.com", "password": "<a password>", "setup_token": "<the token>"}'
```

A success answers `201` with the new `user` and a `token`. The password has to meet the
default policy: at least 12 characters, with an upper-case letter, a lower-case letter and a
digit.

</div>
</div>

Signing in to the console exercises the whole path: the proxy, the console, the engine and
Postgres.

## Upgrades

Pull and recreate. The engine applies any new migrations when it starts:

```bash
docker compose pull engine admin
docker compose up -d
```

`latest` follows every release. For a deliberate upgrade, pin both images to a version tag such
as `ghcr.io/lyeve-labs/lyeve-core:0.56.0` and `ghcr.io/lyeve-labs/lyeve-admin:0.22.0`, and change
the tags when you choose to.

Upgrade the engine before the console. A console release can call routes that only the engine
release beside it serves, so a new console in front of an older engine shows errors on those
pages.

### Upgrading to 0.56.0 and console 0.22.0

Console 0.22.0 reads content history from engine 0.56.0, so upgrade the engine first. Read these
before the first start of 0.56.0:

- **Back up the database.** 0.56.0 adds the tables content history keeps, and an engine before
  0.56.0 refuses to start on a database that holds them. The way back is the backup.
- **Choose where content history runs.** A tenant that already holds content keeps history on
  for every content type, and a new tenant starts with it off. Narrow it, or set how long
  versions are kept, under **Settings > Content history**. See
  [Content history](../features/content-history.md).
- **Name the tenants your event prune scripts mean.** A super admin's prune with no
  `tenant_id` now prunes the super admin's own tenant. Send `"all_tenants": true` to prune
  every tenant. See [Event replay](../features/event-replay.md#prune-the-log).
- **Finish second-factor sign-ins in your API clients.** `POST /api/v1/auth/token` answers an
  account with a second factor with a challenge, finished at
  `POST /api/admin/auth/mfa-verify`. See
  [Multi-factor authentication](../features/multi-factor-auth.md#how-it-works).
- **Expect the console to report unhealthy while the engine is down.** The console image's
  health check asks `/readyz`, which answers `503` while the console cannot reach the engine.

Then upgrade the engine, wait until it reports healthy, and upgrade the console:

```bash
docker compose pull engine admin
docker compose up -d --wait engine
docker compose up -d admin
```

### Upgrading to 0.55.0 and console 0.21.0

Console 0.21.0 needs engine 0.55.0 or later, so upgrade the engine first. Read these before the
first start of 0.55.0:

- **Stop every engine replica first.** An older engine deletes and redacts audit entries
  without sealing the hash chain, so the chain breaks wherever it runs beside 0.55.0. Run
  `docker compose stop engine` before you pull.
- **Review your retention policies.** Scheduled retention enforces every tenant's policies from
  0.55.0, and the first pass runs at boot. Every entry a policy has kept past its period is
  deleted or archived then. See [Audit log](../features/audit-log.md#retention-policies-and-legal-holds).
- **Release legal holds before you delete a tenant.** A tenant with an active legal hold can no
  longer be deleted. The delete answers `409` and removes nothing.
- **List your mail relay's addresses.** The relay is checked at every send. If `SMTP_HOST`
  resolves to loopback or a private network, list every address the name resolves to in
  `SMTP_ALLOWED_PRIVATE_NETWORKS`, both `127.0.0.1` and `::1` for `localhost`, or every send
  is refused. See [Email](../features/email.md#send-through-one-relay).
- **Check your backup schedules.** Stored backup and restore test schedules run from 0.55.0.
  Remove any outside trigger first. See
  [Backup and restore](../how-to/backup-and-restore.md#back-up-to-s3).
- **Run one console replica.** Two requests that renew one session on different console
  replicas at the same moment can end that session. If you run several, give the proxy cookie
  affinity.
- **Update your API clients.** Content fields are nullable on
  [GraphQL](../features/graphql-api.md) output, so regenerate typed clients. A refused password
  on [password reset](../features/password-reset.md) answers `422` instead of `400`. Updating an
  entry no longer changes its status, so publish and unpublish through their own routes in
  [API endpoints](../reference/api-endpoints.md).
- **Expect new error alerts.** [Error tracking](../features/error-tracking.md) opens a new alert
  for each panic site after the upgrade. Resolve or ignore the old alerts once their panics
  have come back as new ones.

Then upgrade the engine, wait until it reports healthy, and upgrade the console:

```bash
docker compose stop engine
docker compose pull engine admin
docker compose up -d --wait engine
docker compose up -d admin
```

## Next

- [Production checklist](production-checklist.md): what to confirm before you take traffic.
- [Backup and restore](../how-to/backup-and-restore.md): tenant snapshots and restores.
- [Email](../features/email.md): outbound mail for invitations and password resets.
- [Kubernetes](kubernetes.md): the same images on a cluster.
