---
title: Configuration reference
description: "Look up every engine setting with its default, and set it through an environment variable, a YAML file or the admin console."
sidebar:
  order: 1
---

The engine is configured with environment variables. This page lists the engine's variables,
grouped by what you are setting up, with the default the engine applies. Each feature page lists the
settings of its own feature, and [Feature settings](#feature-settings) says where each one is.

## Start here

Seven variables decide whether the engine starts. Set these first.

| Variable | What it does | Default |
|---|---|---|
| `DATABASE_URL` | Connection string. The scheme picks the database: `postgres://` or `postgresql://`, `mysql://`, or `sqlserver://`. See [supported databases](../deploy/index.md#supported-databases). | required |
| `JWT_SECRET` | Fallback token signing secret, at least 16 characters. Sessions are signed with an Ed25519 key the engine creates at `JWT_KEY_PATH`, but the engine refuses to start without this value. | required |
| `ENCRYPTION_KEY` | Encrypts stored credentials with AES-256-GCM. At least 32 characters and different from `JWT_SECRET`. A new value makes the stored credentials unreadable, so generate it once. | required |
| `APP_ENV` | `production` or `development`. Production refuses a weak configuration, as listed [below](#what-production-refuses). | `production` |
| `LYEVE_AUDIT_HMAC_KEY` | Key for the audit log's tamper-evident chain. Exactly 64 hex characters. | required in production |
| `RATE_LIMIT_RPS` | Requests per second per client address for the global limiter. Unset or `0` turns the limiter off, which production refuses. | unset |
| `SECURE_COOKIE` | `true` marks cookies `Secure`, sends HSTS and redirects plain HTTP to HTTPS. Production refuses any other value. | `false` |

A minimal production environment, with each secret generated separately:

```bash
DATABASE_URL="postgres://lyeve:<password>@db.example.com:5432/lyeve?sslmode=require"
JWT_SECRET="$(openssl rand -hex 32)"
ENCRYPTION_KEY="$(openssl rand -hex 32)"
LYEVE_AUDIT_HMAC_KEY="$(openssl rand -hex 32)"
RATE_LIMIT_RPS=100
SECURE_COOKIE=true
LYEVE_CONSOLE_URL="https://admin.example.com"
```

For a local try, set `APP_ENV=development` and the production checks are skipped. A
[stateless](../deploy/stateless-mode.md) engine needs neither `DATABASE_URL`, `JWT_SECRET` nor
`ENCRYPTION_KEY`.

### What production refuses

With `APP_ENV` unset or `production`, the engine does not start when:

- `SECURE_COOKIE` is not `true`.
- `RATE_LIMIT_RPS` is unset or `0`.
- `LYEVE_AUDIT_HMAC_KEY` is unset or is not exactly 64 characters.
- `JWT_EXPIRY_SECS` is above `3600`.
- `DATABASE_URL` or `DATABASE_REPLICA_URL` signs in as the database's default superuser:
  `postgres` on PostgreSQL, `root` on MySQL, `sa` on SQL Server.
- The signing key file at `JWT_KEY_PATH` is readable by group or others, or cannot be created.
- `LYEVE_CONSOLE_URL` is set and does not start with `https://`.

In every environment, the engine does not start when:

- `ENCRYPTION_KEY` is unset. The engine then uses `JWT_SECRET` in its place, and the two may
  not be equal.
- `JWT_SECRET` is shorter than 16 characters, or `ENCRYPTION_KEY` shorter than 32, or the two
  are equal.
- Either secret is a placeholder: `secret`, `password` or `default`, or a value containing
  `changeme`, `change-me`, `change-this`, `replace-me`, `replaceme`, `your-secret`,
  `dev-secret`, `insecure`, `example` or `placeholder`.
- `CORS_ORIGINS` contains `*`.
- `JWT_ALG`, `PASSWORD_HASH_ALGO`, `STORAGE_DRIVER` or `DATABASE_DRIVER` names a value the
  engine does not support.
- One of the engine's numbers or durations does not parse. A few, such as the `POOL_*` and
  `HEALTH_*` values, fall back to their default instead.

A production engine starts but logs a warning on every boot when `LYEVE_CONSOLE_URL` is unset,
and when a PostgreSQL `DATABASE_URL` sets no `sslmode` or one other than `require`,
`verify-ca` or `verify-full`.

## Where settings come from

The engine reads each setting from four places. The highest one that has it wins:

1. **An environment variable.** A variable set to an empty value turns the setting off, and no
   lower layer fills it back in.
2. **The YAML file**, `lyeve.yaml`. It wins over the admin console unless the value is tagged
   `!overridable`.
3. **The admin console**, which holds what a feature's configuration form saved.
4. **The built-in default.**

The engine reads the variables in the engine tables on this page once, at startup, before it
connects to the database. Set them in the environment or the file, and restart to change them.
A feature's settings, such as the mail relay or the idempotency TTLs, can also be saved in the
console. A setting the feature reads each time it uses it applies at once. One it reads only
when it starts applies at the next restart. Three are kept until a restart on purpose, because
changing them under a running server breaks something that cannot be undone:

- the WebAuthn relying party (every registered passkey stops working)
- the encryption key (stored credentials become unreadable)
- the idempotency keyspace and backend (a retry stops finding its record, so the request runs
  twice)

**Settings > Configuration** in the admin console shows every setting with its value and the
layer it came from, and saves into the console layer:

![The Configuration page: counts of settings from the environment, the files and the console, and each setting with its value and source.](../how-to/images/configuration.png)

`GET /api/admin/config` (super admin) lists every setting that the environment, the file or the
console sets, with the layer it came from and, for an engine setting, what it does and its
default. Secrets show no value. `PUT /api/admin/config` (super admin) saves into
the console layer and refuses a key a higher layer holds:

```bash
curl -X PUT https://admin.example.com/api/admin/config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"values": {"flow_run_retention": "168h"}}'
```

The answer lists what it stored under `saved`, each with the layer it now resolves from, and
what it refused under `refused`, each with the reason and, for a file value, the file and line. When nothing
could be saved it answers `409`. The [quickstart](../getting-started/quickstart.md) shows how
to get `TOKEN`.

## The YAML file

Every variable that is not marked "environment only" below has a place in `lyeve.yaml`. A
nested key is the variable name with the path joined by underscores, so `storage.s3.bucket` is
`STORAGE_S3_BUCKET`:

```yaml
database:
  url: ${DATABASE_URL}
  max_connections: 25
storage:
  driver: s3
  s3:
    bucket: media
    region: eu-west-1
```

Flat names work too, so an existing `.env` file can be copied in a line at a time. `${VAR}` and
`${VAR:-fallback}` read the process environment, which is how a container passes a secret in
without it living in a committed file.

The engine looks for `lyeve.yaml` or `lyeve.yml` in the working directory, then in
`/etc/lyeve/`, then for a `/etc/lyeve/conf.d/` directory. `LYEVE_CONFIG` points at a specific
file or directory instead. A named path that does not exist stops the boot.

### Feature settings in the file

Feature settings go under a `plugins` block, which drops its own segment, because a feature's
key already names it. `plugins.cache.driver` is `CACHE_DRIVER`:

```yaml
plugins:
  cache:
    driver: !overridable redis
    ttl: "60s"
```

`!overridable` lets the admin console take a key over. Without it the file value is final, and
the console shows the field read-only, naming the file and line it came from.
`LYEVE_OVERRIDABLE` does the same for environment variables: a comma-separated list of keys the
console may take over. A listed key keeps its variable's value until something is saved against
it. It is read once at startup.

### Composing files

```yaml
$include:
  - base.yaml
  - environments/staging.yaml
  - schemas/*.yaml
```

Included files merge first, so the including file overrides them. A `conf.d/` directory beside
the named file merges last, which lets a deployment extend a packaged base without editing it.
A directory of files merges in lexical order, so `10-base.yaml` then `20-staging.yaml` layers
the way it reads. Content types can be declared in these files too. See
[Moving content types between projects](content-type-portability.md#declaring-content-types-in-configuration).

### Credentials saved in the console

A credential saved on a feature's configuration form is encrypted at rest with a key derived
from `ENCRYPTION_KEY` and is never returned. A set credential reads as `********`. Writing that
value back leaves the stored one alone, so editing one field of a form does not overwrite the
rest. Submitting an empty value clears it.

## Database

| Variable | What it does | Default |
|---|---|---|
| `DATABASE_URL` | See [Start here](#start-here). | required |
| `DATABASE_MAX_CONNECTIONS` | Most connections the engine opens to the database. Multiply by the number of replicas and stay under the database's own limit. | `25` |
| `DATABASE_REPLICA_URL` | A read replica. Read-heavy queries that tolerate lag go to it. Production refuses a default superuser here too. | unset (all reads go to the primary) |
| `DATABASE_REPLICA_MAX_CONNS` | Most connections to the replica. | `30` |

<details>
<summary>Pool tuning and an external pooler</summary>

| Variable | What it does | Default |
|---|---|---|
| `DB_POOL_MIN_CONNS` | Connections opened at startup, so the first requests do not wait for a handshake. | `2` |
| `DB_CONN_MAX_LIFETIME` | How long a connection is used before it is replaced, as a duration. | `1h` |
| `DB_POOL_HEALTH_CHECK_PERIOD` | How long an idle connection stays open before it is closed. Above zero, it replaces `DB_CONN_MAX_IDLE_TIME`. | `30s` |
| `DB_CONN_MAX_IDLE_TIME` | The idle timeout used when `DB_POOL_HEALTH_CHECK_PERIOD` is `0`. | `5m` |
| `MIGRATIONS_PATH` | Directory of the database migrations the engine applies at startup. The image sets it. | `./migrations` (`/app/migrations` in the image) |
| `DATABASE_DRIVER` | Overrides the database type read from `DATABASE_URL`: `postgres`, `mysql` or `mssql`. Leave it unset. | read from the URL |
| `CONNECTION_POOLER` | `pgbouncer` or `proxysql` when an external pooler sits in front of the database, so the pool health report at `GET /api/admin/pool/health` includes its sizing. The engine does not configure the pooler. | `none` |
| `POOL_MAX_CLIENT_CONN` | The pooler's client connection limit, for that report. Environment only. | `100` |
| `POOL_DEFAULT_POOL_SIZE` | The pooler's default pool size, for that report. Environment only. | `20` |
| `POOL_RESERVE_POOL_SIZE` | The pooler's reserve pool size, for that report. Environment only. | `5` |
| `POOL_TENANT_SIZES` | Per-tenant pool sizes as a JSON array of `{"slug", "pool_size", "min_pool_size", "max_connections", "db_name"}`. Invalid JSON is logged and ignored. Environment only. | unset |
| `POOL_HEALTH_MAX_LATENCY` | Ping latency above which the pool health report says degraded. Environment only. | `1s` |
| `POOL_HEALTH_MIN_IDLE` | Idle connections below which the report says degraded. Environment only. | `1` |
| `POOL_HEALTH_MAX_UTIL` | Share of the pool in use above which the report says degraded. Environment only. | `0.9` |

The `CONNECTION_POOLER` and `POOL_*` sizing variables are read only when `CONNECTION_POOLER`
names a pooler. The `POOL_HEALTH_*` thresholds are read on every install.

</details>

## Network and proxies

| Variable | What it does | Default |
|---|---|---|
| `LYEVE_CONSOLE_URL` | The admin console's public URL, such as `https://admin.example.com`. Password reset and magic-link emails link to pages the console serves, and device sign-in shows its approval page there. It must be an absolute URL with no query or fragment. In production it must be `https`, and while it is unset password reset and magic-link sign-in refuse to start and device sign-in answers `503`. | `http://localhost:5173` outside production |
| `LYEVE_BASE_URL` | The public base URL of the engine. Features that build an absolute link to it use this value instead of the request's `Host` header: OAuth and SAML callback URLs, data export downloads and file links. Set it in production. | unset |
| `LYEVE_OUTBOUND_ALLOW_CGNAT` | `true` lets webhooks, flows, identity providers, JWKS fetches and alert channels reach the shared address space `100.64.0.0/10`, which Tailscale hands out, and the NAT64 or 6to4 form of an address in it. It opens the range to every tenant's outbound targets, cloud metadata services in that range included. Email honors it too: an SMTP relay or API provider host in that range needs no `SMTP_ALLOWED_PRIVATE_NETWORKS` entry. Read once at boot, and the admin console cannot set it. See [where outbound requests may go](../trust/security-controls.md#where-outbound-requests-may-go). | `false` |
| `TRUSTED_PROXIES` | Comma-separated CIDRs of the reverse proxies in front. `X-Forwarded-For` is honored only from these, so rate limits and audit entries see the real client address. Unset, every request counts as the proxy's. | unset |
| `CORS_ORIGINS` | Comma-separated origins of the browser apps that call the APIs cross-origin. `*` is refused. | `http://localhost:5173` |
| `ADMIN_LISTEN_ADDR` | Bind address of the Admin API. | `0.0.0.0:3001` |
| `API_LISTEN_ADDR` | Bind address of the Content API. | `0.0.0.0:3002` |
| `TLS_CERT_FILE`, `TLS_KEY_FILE` | A PEM certificate and key, set together. Both listeners then serve TLS 1.2 or later and pick up a rotated certificate without a restart. Unset, both serve plain HTTP for a proxy that terminates TLS. | unset |

<details>
<summary>Host and address filters, CORS details</summary>

| Variable | What it does | Default |
|---|---|---|
| `ALLOWED_HOSTS` | Comma-separated host names the engine answers to. Any other `Host` header gets `421`. | unset (every host) |
| `IP_ALLOWLIST` | Comma-separated CIDRs or addresses allowed to connect. Others get `403`. An entry that does not parse stops the boot. | unset (every address) |
| `CORS_ALLOWED_DOMAINS` | Comma-separated domain suffixes under which a tenant's own domain may call the APIs cross-origin, such as `example.com` for `shop.example.com`. Empty allows no tenant domain. | unset |
| `CORS_ALLOW_METHODS` | Methods the preflight allows. | `GET, POST, PUT, DELETE, PATCH, OPTIONS` |
| `CORS_ALLOW_HEADERS` | Request headers a browser may send. It replaces the default list, so keep `Authorization` and `X-Tenant-ID` if you set it. | `Content-Type, Authorization, X-API-Key, X-Tenant-ID, X-Correlation-ID, X-CSRF-Token` |
| `CORS_EXPOSE_HEADERS` | Response headers a cross-origin script may read. The default lets a browser client read the request id and the rate-limit values. | `X-Request-Id, X-Correlation-ID, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, Retry-After` |
| `CORS_MAX_AGE` | Seconds a browser may cache the preflight. | `3600` |

</details>

## Sign-in, tokens and passwords

| Variable | What it does | Default |
|---|---|---|
| `JWT_KEY_PATH` | Where the Ed25519 signing key is kept. Put it on a volume, or give every replica the same file, or each restart signs everyone out. The key alone does not keep sessions through a restart: refresh tokens live in the engine's memory unless `CACHE_DRIVER=redis` and `REDIS_URL` point at a Redis. In production the file must be readable by its owner only. | `/var/lib/lyeve/jwt_key.json` |
| `JWT_EXPIRY_SECS` | Session token lifetime in seconds. Production refuses a value above `3600`. | `900` (15 minutes) |
| `REFRESH_TOKEN_TTL_SECS` | Refresh token lifetime in seconds. | `2592000` (30 days) |
| `LYEVE_SETUP_TOKEN` | The token the first-run setup asks for, at least 16 characters. Unset, the engine prints a one-time token to its log while no account exists. Set it when more than one replica can answer setup, since each prints its own. | unset (a logged one-time token) |
| `ADMIN_CONSOLE_KEY` | A secret of at least 32 characters, shared with the admin console. The console signs its calls with the browser's address, so sign-in limits count each person instead of the console server. | unset |
| `PASSWORD_MIN_LENGTH` | Shortest password accepted. | `12` |

<details>
<summary>Signing, rotation, password rules and setup mode</summary>

| Variable | What it does | Default |
|---|---|---|
| `JWT_ALG` | `EdDSA` signs sessions with the Ed25519 key. `HS256` signs them with `JWT_SECRET` and creates no key file. Any other value stops the boot. Environment only: a value in the file passes the check but does not change the signing. | `EdDSA` |
| `JWT_SECRETS` | Comma-separated HS256 secrets that verify tokens during a rotation. `JWT_SECRET` still signs, so list its value first and the old secret after it. Unset, `JWT_SECRET` alone verifies. | unset |
| `ADMIN_CONSOLE_KEY_PREVIOUS` | The old console key, still accepted while you rotate. Set the new key in `ADMIN_CONSOLE_KEY`, move the console to it, then unset this one. Needs `ADMIN_CONSOLE_KEY`, at least 32 characters. | unset |
| `PASSWORD_HASH_ALGO` | `bcrypt` or `argon2id`. | `bcrypt` |
| `PASSWORD_REQUIRE_COMPLEXITY` | Requires an upper-case letter, a lower-case letter and a digit. | `true` |
| `PASSWORD_CHECK_COMMON` | Refuses passwords on a list of common ones. | `true` |
| `TRUSTED_ISSUERS` | Comma-separated base URLs of outside OpenID Connect issuers whose tokens the Content API accepts. Keys are fetched from `{issuer}/.well-known/jwks.json`. See [Scaling](../scaling/index.md#tokens-from-other-issuers). | unset |
| `TRUSTED_ISSUER_POLICIES` | A JSON array giving each trusted issuer its tenant and the roles its users map to. An issuer with no policy has its tokens ignored, and a policy for an unlisted issuer stops the boot. | unset |
| `API_KEY_PEPPER` | A server secret mixed into every stored API key hash. See [API keys](../features/api-keys.md). | unset |
| `ENFORCE_API_KEY_PEPPER` | `true` stops the boot when no `API_KEY_PEPPER` is found. | `false` |
| `LYEVE_SETUP_MODE` | `true` lets the engine start without `DATABASE_URL`, `JWT_SECRET` or `ENCRYPTION_KEY`. It then serves only the probes and the setup routes, and the admin console shows what to set. See [Installation](../getting-started/installation.md). | unset |

</details>

## Request limits

| Variable | What it does | Default |
|---|---|---|
| `RATE_LIMIT_RPS` | See [Start here](#start-here). | unset |
| `RATE_LIMIT_BURST` | Requests a client address may send at once above the steady rate. | twice `RATE_LIMIT_RPS`, at least `1` |
| `MAX_BODY_BYTES` | Largest request body on a route that sets no limit of its own, in bytes. Sized for uploads. A route that sets its own limit keeps it whatever this is, such as [bulk import](../features/bulk-import.md#a-file-over-50-mib) at 70,953,644 bytes. | `10485760` (10 MiB) |
| `MAX_JSON_BODY_BYTES` | Largest body on JSON routes. A request without `Content-Length`, such as a chunked one, is cut off once it passes the same size. | `1048576` (1 MiB) |

<details>
<summary>Per-tenant limits, public routes and load shedding</summary>

| Variable | What it does | Default |
|---|---|---|
| `RATE_LIMIT_PER_TENANT` | `true` counts the global limit per tenant and address instead of per address. | `false` |
| `PUBLIC_RATE_LIMITS` | Per-route caps for public routes, comma-separated `METHOD:/pattern=rate:burst` entries using the route pattern exactly as documented, such as `POST:/api/v1/flows/hooks/{flow_id}=20:40`. Entries merge over the built-in table, so a route you do not name keeps its own limit. A malformed entry stops the boot and names the entry. | built-in table |
| `PUBLIC_RATE_LIMIT_GLOBAL` | Combined per-address cap across every public route, as `rate:burst`. Raise it where many callers share one address: a CDN, a proxy on loopback, a load generator. | `50:100` |
| `BACKPRESSURE_ENABLED` | Turns load shedding on. See [Scaling](../scaling/index.md). | `false` |
| `BACKPRESSURE_MAX_INFLIGHT` | Requests in flight before the engine sheds load. | `200` |
| `BACKPRESSURE_TENANT_QUOTA_PCT` | Share of that budget one tenant may hold, above `0` and at most `1`. | `0.4` |
| `BACKPRESSURE_POOL_PRESSURE_THRESHOLD` | Share of the database pool in use at which shedding begins, above `0` and at most `1`. | `0.85` |

To share the per-address limit across replicas through Redis, set `RATE_LIMIT_BACKEND=redis`.
That setting belongs to [Rate limiting](../features/rate-limiting.md).

</details>

## Storage

These settings drive the engine's own file store. The [Object storage](../features/object-storage.md)
feature runs on every install by default and takes its place. While it runs, `STORAGE_DRIVER`
changes nothing, and the `STORAGE_S3_*` values only create its first provider on a start that
finds none.

| Variable | What it does | Default |
|---|---|---|
| `STORAGE_DRIVER` | Where uploaded files go: `local` or `s3`. | `local` |
| `STORAGE_LOCAL_PATH` | Directory for the `local` driver. Mount a volume here. | `./uploads` (`/app/uploads` in the image) |
| `STORAGE_BASE_URL` | Public base URL of files kept by the `local` driver. | unset |

<details>
<summary>S3-compatible bucket</summary>

| Variable | What it does | Default |
|---|---|---|
| `STORAGE_S3_BUCKET` | Bucket name. | unset |
| `STORAGE_S3_REGION` | Bucket region. | `us-east-1` |
| `STORAGE_S3_ENDPOINT` | Endpoint of an S3-compatible service such as R2 or MinIO. Unset uses AWS. | unset |
| `STORAGE_S3_KEY` | Access key id. | unset |
| `STORAGE_S3_SECRET` | Secret access key. | unset |
| `STORAGE_S3_CDN_BASE_URL` | Public base URL that file links use, such as a CDN in front of the bucket. | unset |
| `STORAGE_S3_USE_SSL` | Connects to the endpoint over TLS. | `true` |
| `STORAGE_S3_FORCE_PATH_STYLE` | `true` addresses the bucket in the path, as MinIO needs. | `false` |
| `STORAGE_S3_MULTIPART_MB` | Size in MB above which an upload is sent in parts. | `5` |

[Object storage](../features/object-storage.md) covers buckets per provider and signed links.

</details>

## Email

The mail relay is read by the features that send mail: [Email](../features/email.md),
[Password reset](../features/password-reset.md) and
[Magic link sign-in](../features/magic-link-sign-in.md). It can also be saved on the email
feature's configuration form, and a variable set in the deployment still wins.

| Variable | What it does | Default |
|---|---|---|
| `SMTP_HOST` | SMTP server host name. Unset, no mail is sent. | unset |
| `SMTP_PORT` | SMTP server port. | `587` |
| `SMTP_USER` | SMTP user name. | unset |
| `SMTP_PASS` | SMTP password. | unset |
| `SMTP_FROM` | The `From` address of outbound mail. | unset |
| `SMTP_TLS` | Exactly `true` selects implicit TLS, usually on port 465. Any other value sends over STARTTLS. Read by the Email feature only. | `false` |

## Caching and Redis

| Variable | What it does | Default |
|---|---|---|
| `CACHE_TTL` | How long the Content API keeps an entry or a list in its memory cache, as a duration. The [Caching](../features/caching.md) feature uses it as its default TTL. | `60s` |
| `CACHE_MAX_ENTRIES` | Most entries in each of those memory caches. | `1000` |
| `CACHE_DRIVER` | The Caching feature's backend: `memory` or `redis`. Set `redis` to share sessions and sign-in lockouts across replicas. The Content API caches stay in memory either way. | `memory` |
| `REDIS_URL` | The Redis that caching, idempotency, shared rate limits, realtime and flows use, unless a feature names its own. | `redis://localhost:6379/0` |

[Scaling](../scaling/index.md) lists what more than one replica needs from Redis.

## Idempotency

An `Idempotency-Key` header on a write makes it safe to retry: the request runs once, and a
retry with the same key gets the stored answer. Keys are kept per tenant and user. Included free.
See [Idempotent requests](../features/idempotent-requests.md).

| Variable | What it does | Default |
|---|---|---|
| `REDIS_URL` | Shares idempotency records between replicas, so a retry that lands on another replica still finds its record. A Redis on the local host that does not answer falls back to in-process records and logs why. A remote one that does not answer stops the feature from starting. | `redis://localhost:6379/0` |
| `IDEMPOTENCY_KEYSPACE` | Prefix on every stored key. Needs a restart to change, as does the backend. The settings below apply without one. | `idem` |
| `IDEMPOTENCY_LOCK_TTL` | How long a request in flight holds its key. A handler that outlives its lock is logged. | `30s` |
| `IDEMPOTENCY_DATA_TTL` | How long a finished answer is kept for replay. | `24h` |
| `IDEMPOTENCY_AUTO_GENERATE` | Derives a key from the tenant, user, method, path and body when a request has none. | `false` |
| `IDEMPOTENCY_AUTO_METHODS` | Comma-separated methods that get a derived key. | `POST` |

## License, features and mode

| Variable | What it does | Default |
|---|---|---|
| `LYEVE_LICENSE_KEY` | A signed license token, verified offline, or a license key, which the engine exchanges once for a token. Free features run with or without it. | unset (free features only) |
| `LYEVE_PLUGINS` | Comma-separated features to start. Empty starts every feature that is free or your license covers. See [Licensing and tiers](../concepts/licensing-and-tiers.md). | empty |
| `LYEVE_LICENSE_SERVER_URL` | Where a license key is exchanged. `https` only. Never contacted when `LYEVE_LICENSE_KEY` holds a token. | `https://api.lyeve.com` |
| `LYEVE_LICENSE_CACHE_DIR` | Where the verified token and the install id are cached. | `/var/lib/lyeve` |
| `MULTI_TENANT` | `true` serves more than one tenant from one install. See [Tenants](../features/tenants.md). | `false` |
| `LYEVE_MODE` | `stateless` runs the engine with no database. Any other value stops the boot. See [Stateless mode](../deploy/stateless-mode.md). | unset |
| `LYEVE_CONFIG` | Path of the YAML file or directory to read. Environment only. | unset (the search above) |
| `LYEVE_OVERRIDABLE` | Comma-separated variables the admin console may take over. Environment only. | unset |

## Health checks and shutdown

`/healthz`, `/readyz` and `/startup` answer on both listeners. See
[health endpoints](../deploy/docker.md#health-endpoints) for what each one checks.

| Variable | What it does | Default |
|---|---|---|
| `PRESTOP_DRAIN_SECS` | Seconds `/readyz` answers `503` before shutdown starts, so the load balancer stops sending traffic first. | `5` |
| `GRACEFUL_SHUTDOWN_SECS` | Seconds requests in flight get to finish. | `60` |

<details>
<summary>Readiness thresholds</summary>

| Variable | What it does | Default |
|---|---|---|
| `HEALTH_DISK_PATH` | Directory whose free space `/readyz` checks. Environment only. | `STORAGE_LOCAL_PATH` |
| `HEALTH_DISK_MIN_MB` | Free space in MB below which `/readyz` answers `503`. Environment only. | `100` |
| `HEALTH_REDIS_PROBE_REQUIRED` | `true` makes `/readyz` answer `503` when the Redis that holds sessions is unreachable. Unset, that outage is reported but the replica stays in service. Environment only. | `false` |
| `HEALTH_GOROUTINE_MAX` | Goroutine count above which `/readyz` reports a warning. It does not fail the check. Environment only. | `10000` |
| `POOL_EXHAUSTION_THRESHOLD` | Share of the database pool in use above which `/readyz` reports a warning. It does not fail the check. Environment only. | `0.8` |

</details>

## Observability

| Variable | What it does | Default |
|---|---|---|
| `METRICS_TOKEN` | Bearer token for `GET /api/admin/metrics`, so a scraper needs no admin account. While it is set, any other bearer token on that route answers `401`. Unset, the endpoint answers a super admin. | unset |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OpenTelemetry collector to send traces to. Unset, tracing is off. | unset |
| `INSTANCE_ID` | Names this replica in logs, traces and the replica list. | host name and process id |

<details>
<summary>Tracing, debug traces and compression</summary>

| Variable | What it does | Default |
|---|---|---|
| `OTEL_EXPORTER_OTLP_INSECURE` | `true` sends to the collector without TLS. | `false` |
| `TRACING_SAMPLING_RATE` | Share of traces kept, `0.0` to `1.0`. A value outside that range stops the boot. | `1.0` |
| `TRACING_TENANT_SAMPLING_RATES` | Per-tenant rates as comma-separated `tenant=rate` pairs. A malformed pair stops the boot. | unset |
| `LYEVE_DEBUG_TRACER_ENABLED` | `true` lets an admin send `X-Debug: true` and get a trace of the request instead of the normal answer. | `false` |
| `COMPRESS_GZIP_JSON_LEVEL` | gzip level for JSON responses. Environment only. | `4` |
| `COMPRESS_BROTLI_JSON_LEVEL` | Brotli level for JSON responses. Environment only. | `2` |
| `COMPRESS_GZIP_TEXT_LEVEL` | gzip level for text and HTML responses. Environment only. | `6` |
| `COMPRESS_BROTLI_TEXT_LEVEL` | Brotli level for text and HTML responses. Environment only. | `4` |

</details>

## Privacy and regions

| Variable | What it does | Default |
|---|---|---|
| `LYEVE_GDPR_HOLD_CHECK` | What an erasure request does when a legal-hold check is set up but cannot answer. `refuse` answers `503` and erases nothing. `proceed` erases anyway. Environment only. | `refuse` |
| `INSTANCE_REGION` | The data region this install serves. Answers carry it in `X-CMS-Region`, and [Data residency](../features/data-residency.md) compares a tenant's region against it. | unset |

An active legal hold stops `POST /api/admin/gdpr/erase` from erasing the person it covers. The
request succeeds with `total_rows: 0` and names the holds. Holds need the `audit-retention`
feature. See [Data protection](../trust/data-protection.md).

## Concurrency and resources

The engine sizes itself to the container's memory and CPU limits, which it reads from the
cgroup. `GOMEMLIMIT` and `GOMAXPROCS`, Go's own variables, win when you set them.

<details>
<summary>Memory, CPU and worker pools</summary>

| Variable | What it does | Default |
|---|---|---|
| `MEMORY_LIMIT_BYTES` | Memory limit used when the cgroup sets none. The engine sets Go's soft memory limit to 85% of the limit it finds. | from the cgroup |
| `CPU_QUOTA` | Whole cores the engine schedules for, instead of the cgroup's CPU limit. | from the cgroup |
| `GOROUTINE_ENGINE_ENABLED` | Warms feature caches and stops features in parallel, instead of one at a time. The shared worker pool runs either way. Needs a license with `goroutine-engine-pro`, and without it the variable is ignored with a warning in the log. See [Concurrency tuning](../features/concurrency-tuning.md). | `false` |
| `GOROUTINE_ENGINE_POOL_SIZE` | Workers in the shared pool. Needs a license with `goroutine-engine-pro`, and without it the pool keeps 100 workers. | `100` |
| `DB_WARMUP_PARALLELISM` | How many tasks the parallel stages, such as cache warmup, run at once. | `4` |
| `ASYNC_HOOKS_ENABLED` | Runs after-save hooks in the background instead of inside the request. Needs a license with `goroutine-engine-pro`, and without it hooks run in order inside the request. | `false` |
| `ASYNC_HOOK_TIMEOUT` | How long one asynchronous hook may run. Needs a license with `goroutine-engine-pro`. | `5s` |
| `CONTROL_STRICT_MODE` | `true` stops the boot when a security control reports that it is not enforced. Unset, the engine logs the report and starts. Environment only. | `false` |

</details>

## Feature settings

Each feature lists its own settings on its page. Find a variable here by its prefix.

| You are setting up | Variables | Page |
|---|---|---|
| Shared caching | `CACHE_*`, `REDIS_URL` | [Caching](../features/caching.md) |
| Rate limits across replicas | `RATE_LIMIT_BACKEND`, `RATE_LIMIT_REDIS_URL` | [Rate limiting](../features/rate-limiting.md) |
| Uploads and images | `MEDIA_*` | [Media](../features/media.md) |
| Signed upload and download links | `MAX_PRESIGN_*`, `STORAGE_LOCAL_SIGNING_SECRET` | [Object storage](../features/object-storage.md) |
| Keeping a tenant's files in its region without a data residency assignment | `STORAGE_TENANT_REGIONS` | [Object storage](../features/object-storage.md) |
| Search | `SEARCH_*` | [Search](../features/search.md) |
| Outbound mail | `SMTP_*` | [Email](../features/email.md) |
| Magic links | `MAGIC_LINK_*` | [Magic link sign-in](../features/magic-link-sign-in.md) |
| Passkeys and one-time codes | `WEBAUTHN_*`, `MFA_TOTP_ISSUER` | [Multi-factor authentication](../features/multi-factor-auth.md) |
| Bot checks | `CAPTCHA_*` | [Captcha](../features/captcha.md) |
| Device and IP reputation | `DEVICE_FINGERPRINT_CONFIG`, `IPREP_*` | [Trusted devices](../features/trusted-devices.md) |
| GraphQL | `GRAPHQL_*` | [GraphQL API](../features/graphql-api.md) |
| gRPC | `GRPC_*` | [gRPC API](../features/grpc-api.md) |
| Realtime | `REALTIME_*` | [Realtime](../features/realtime.md) |
| Message brokers | `EVENT_BUS`, `NATS_*`, `KAFKA_*`, `RABBITMQ_*` | [Message broker events](../features/message-broker-events.md) |
| Outbound webhooks | `WEBHOOK_ALLOWED_PRIVATE_NETWORKS` | [Webhooks](../features/webhooks.md) |
| Flows | `FLOW_RUN_RETENTION`, `FLOW_CRON_ENABLED` | [Flows](../features/flows.md) |
| Metrics export | `METRICS_*` other than `METRICS_TOKEN` | [Metrics export](../features/metrics-export.md) |
| Error alerts | `ERROR_TRACKING_*`, `SLACK_WEBHOOK_URL`, `DISCORD_WEBHOOK_URL` | [Error tracking](../features/error-tracking.md) |
| Analytics | `ANALYTICS_*` | [Analytics](../features/analytics.md) |
| Uptime probes | `SYNTHETIC_MONITORING_*` | [Synthetic monitoring](../features/synthetic-monitoring.md) |
| Request profiling | `PROFILER_*` | [Request profiling](../features/request-profiling.md) |
| Slow queries | `QUERY_MONITOR_*` | [Slow query analysis](../features/slow-query-analysis.md) |
| Data exports | `DATA_EXPORT_*` | [Data export](../features/data-export.md) |
| Tenant backups and keys | `KMS_*`, `STORAGE_S3_BACKUP_ALLOWED_BUCKETS`, `REPLICATION_TARGETS` | [Tenants](../features/tenants.md) |
| AI | `AI_*` | [AI](../features/ai.md) |

## Related

- [Production checklist](../deploy/production-checklist.md): every setting to confirm before
  go-live, one line each.
- [Secure your instance](../how-to/secure-your-instance.md): the hardening steps behind these
  settings.
- [Scaling](../scaling/index.md): what more than one replica needs.
- [Choose where to run LyEve](../deploy/index.md): hosts and databases.
