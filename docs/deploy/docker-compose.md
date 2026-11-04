---
title: Docker Compose
description: "Run the engine, LyEve Console, Postgres and a TLS proxy on one host with one Compose file."
sidebar:
  order: 14
---

This page gives you the whole stack in two files: the engine
(`ghcr.io/lyeve-labs/lyeve-core`), LyEve Console (`ghcr.io/lyeve-labs/lyeve-console`), a
Postgres database and a Caddy proxy that obtains TLS certificates. Copy both files, write the
`.env`, and run `docker compose up -d`.

:::tip[Best for]
- **Production on one server**, with the console and TLS included.
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
    image: ghcr.io/lyeve-labs/lyeve-console:latest
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

# The console. The browser also calls the engine's APIs on this origin.
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

The engine writes the token to its standard error, not to its log. Its log carries a line that
says where the token went, without the token.

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
as `ghcr.io/lyeve-labs/lyeve-core:0.58.0` and `ghcr.io/lyeve-labs/lyeve-console:0.25.0`, and change
the tags when you choose to.

Upgrade the engine before the console. A console release can call routes that only the engine
release beside it serves, so a new console in front of an older engine shows errors on those
pages.

### Upgrading to console 0.25.0

Console 0.25.0 is published as `ghcr.io/lyeve-labs/lyeve-console`, so change the console's
`image:` line to that name. Every 0.25 release is also published under the old name,
`ghcr.io/lyeve-labs/lyeve-admin`, with the same tags and the same digests, so a file that still
names it keeps updating until 0.26.0. The tags already published under the old name stay. The
release changes the console's name and look, and adds no setting:

```bash
docker compose pull admin
docker compose up -d admin
```

### Upgrading to 0.58.0 and console 0.24.0

Upgrade the engine first, then the console. Read these before the first start of 0.58.0:

- **Back up the database.** The first start adds a column to the accounts table and runs new
  migrations for audit, media, storage, search, usage and access rules. On PostgreSQL, search
  rebuilds the rows of each content type a public search rule restricts, in the background.
- **Find the setup token on standard error.** A new install's one-time setup token is written
  to the engine's standard error, outside the log. A script that read `setup_token` from the
  log reads standard error instead, or sets `LYEVE_SETUP_TOKEN`.
- **Check your mail relay.** With implicit TLS off, through `SMTP_TLS` or the **TLS** switch on
  an SMTP provider, a send to a relay that does not offer STARTTLS fails. Before you upgrade,
  turn on implicit TLS, usually on port 465, or use a relay that offers STARTTLS. See
  [Email](../features/email.md).
- **Choose how SQL Server deletes a tenant.** On SQL Server a tenant delete removes the tenant's
  content rows in batches. Set `SCHEMA_MSSQL_PURGE_MODE=single` to keep the earlier behavior.
  The first start builds a tenant index on every relation table made before.
- **Decide how long slow query captures stay.** Captures older than 30 days are deleted every
  hour. Set `query_monitor_retention_days` to zero or less to keep them all.
- **Check your quota grace periods.** A tenant past a hard limit with blocking on is now blocked
  only once its grace period has passed, and the API sets it to 24 hours by default. Set the
  grace period to `0` for an immediate block. See
  [Tenant quotas](../features/tenant-quotas.md).
- **Update your directory and API clients.** A SCIM `PUT` keeps the attributes it leaves out,
  so send an attribute as null or empty to clear it. Saving a draft on a content type without
  draft and publish answers `422` with `VALIDATION_NO_DRAFTS`. Enabling or deleting an account
  whose erasure has not completed answers `409` with `USER_ERASURE_PENDING` until the erasure
  completes.
- **Filter the audit log by the digests the engine gives you.** Entries written from 0.58.0
  name the subject of an export, an erasure or a refused sign-in by a keyed digest. A tool that
  filters by a SHA-256 it computes finds only the older entries, so ask
  `POST /api/admin/gdpr/subject-digests` for every value to filter by.
- **Keep content with bulk import.** Bulk import refuses to start without the content feature,
  so an install that limits its features keeps both.
- **Turn history off before a downgrade.** On a content type with history on, a list of objects
  is stored one element at a time. An older release refuses such a version, and an edit of that
  entry, while history is on for its type.
- **Set `SIGN_IN_ORGANIZATION` where a console needs it.** From console 0.24.0 the sign-in,
  password reset and email link pages ask for an organization only when the console runs with
  `SIGN_IN_ORGANIZATION=true`. A console that serves several organizations on an address that
  names none sets it, or its sign-in pages stop asking.

Then upgrade the engine, wait until it reports healthy, and upgrade the console:

```bash
docker compose stop engine
docker compose pull engine admin
docker compose up -d --wait engine
docker compose up -d admin
```

### Upgrading to 0.57.0 and console 0.23.0

Upgrade the engine first, then the console. Read these before the first start of 0.57.0:

- **Back up the database.** The first start indexes every content table and, on MySQL and SQL
  Server, rebuilds the accounts table. The backup is the way back.
- **On MySQL or SQL Server, stop every engine first.** The first start rewrites every stored
  account address into one Unicode form and rebuilds the accounts table, so an older engine
  must not run beside it. On a test host the rebuild took about 18 seconds per million accounts
  on MySQL, where account writes wait for it, and about 3 seconds per million on SQL Server,
  where reads and writes both wait. Start the new release on every instance together.
- **Let the first start finish.** If it is stopped during the rebuild, the database rolls the
  change back and the next start refuses with `Dirty database version 12`. Set the recorded
  version back with `UPDATE engine_schema_migrations SET version = 11, dirty = false`
  (`dirty = 0` on SQL Server) and start again.
- **Upgrade large instances in a quiet window.** The first start builds an index on every
  content table, one table at a time, while the engine serves. Writes to a table pause while its
  index builds, from milliseconds on PostgreSQL to about 2 seconds on MySQL, and SQL Server
  Standard and Express hold writes for the whole build. Until it ends, saving that content
  type's definition answers `503` with `Retry-After`. The first start also removes duplicate
  rows from the query log and indexes it, which took up to 35 seconds for a million rows on SQL
  Server.
- **Read the first start's log on MySQL or SQL Server.** It names each account that kept its
  address because another account already holds that address. A SCIM record
  whose user name now collides with an earlier record of the same directory is deactivated and
  renamed, the audit log records `scim_user.collision_rename` for it, and single sign-on through
  the directory is refused for that person until the directory renames or removes the record.
- **Update your webhook receivers and flows.** An entry saved in the console, an import, a
  scheduled publish or a release now reaches webhooks, flows and the message broker with each
  field at the top level of `data` and the status under `data._status`, as a Content API write
  does. A receiver that read `data.body.title` reads `data.title`, and on a content type with
  drafts one that read `data.status` reads `data._status`. A flow that reads
  `trigger.data.status` for the publication status, one made from the `chat-notify` or
  `cdn-purge` template included, reads `trigger.data._status`. See
  [Webhooks](../features/webhooks.md).
- **Update your API clients.** `POST /api/admin/users` must name the account's roles, so send
  `"roles": ["editor"]` where a request with no roles used to create an editor. A data subject
  erasure through `POST /api/admin/gdpr/erase` reports `complete`: treat a `207` as unfinished
  and retry a `503`. An account under erasure cannot sign in until its erasure completes. A
  person's own data export answers only a signed-in session, once an hour per account, so move
  an API key or admin token that exported through it to a session or to
  `POST /api/admin/gdpr/export`.
- **Choose what an erasure does with form submissions.** An erasure deletes the files the
  person uploaded and, by default, each form submission that holds only the person's addresses.
  If submissions must stay, send `PUT /api/admin/schemas/form-erasure` with `{"mode": "soft"}`
  as a tenant admin before the next erasure.
- **Replace short flow secrets.** Saving a flow secret variable shorter than 8 bytes is
  refused. A shorter one saved before keeps working but is masked only where it appears whole,
  so replace it with a longer one.
- **Check your public search lists.** Public search on Elasticsearch and Meilisearch now finds
  entries published with the Content API and matches every field. List only content types whose
  fields may all be public. See [Search](../features/search.md#let-anyone-search-your-site).

Then upgrade the engine, wait until it reports healthy, and upgrade the console:

```bash
docker compose stop engine
docker compose pull engine admin
docker compose up -d --wait engine
docker compose up -d admin
```

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
