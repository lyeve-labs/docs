---
title: Kubernetes
description: "Deploy the engine and the admin console on Kubernetes with replicas, a shared signing key, probes and an Ingress."
sidebar:
  order: 15
---

LyEve runs on Kubernetes as two Deployments, one per image:

- `ghcr.io/lyeve-labs/lyeve-core`, the engine: the Admin API on **3001**, and the Content API and
  `/.well-known/jwks.json` on **3002**.
- `ghcr.io/lyeve-labs/lyeve-admin`, the admin console: a Node server on **3002** that calls the
  engine for the browser. It holds no database connection.

:::tip[Best for]
- **Several engine replicas** behind one Ingress, with rolling deploys and no downtime.
- **A cluster you already run**, with its secret store and certificate manager.
:::

The database is external. Point `DATABASE_URL` at PostgreSQL, MySQL 8.0 or 8.4, or SQL
Server, managed or running in the cluster. The manifests below also expect a Redis and an
S3-compatible bucket, because more than one engine replica needs both. To run one replica with neither, see
[One replica](#one-replica).

Every value marked `# cluster-specific` must be replaced before you apply the manifests.

## Topology

| Workload | Image | Ports | Reached through |
|---|---|---|---|
| `lyeve-core` | `ghcr.io/lyeve-labs/lyeve-core` | 3001, 3002 | The Ingress: `/api` on the admin host, everything on the API host |
| `lyeve-admin` | `ghcr.io/lyeve-labs/lyeve-admin` | 3002 | The Ingress: the admin host |

Both Services are `ClusterIP`. Only the Ingress faces the internet. Give port 3001 no public
host or `LoadBalancer` of its own: the admin host routes `/api/admin` to it, which is all the
admin console needs.

## What more than one replica needs

| Shared state | Setting |
|---|---|
| The token signing key | The same key file on every pod, from a Secret. [Below](#the-signing-key) |
| Sessions, sign-in lockouts and MFA limits | `CACHE_DRIVER=redis` and `REDIS_URL`. Without them a refresh that reaches another pod answers `401` and logs the user out |
| The per-IP rate limit | `RATE_LIMIT_BACKEND=redis`. Without it each pod admits the full limit |
| Uploaded media | An S3-compatible bucket, set with `STORAGE_S3_*` before the first start. See [Object storage](../features/object-storage.md) |
| First-run setup | `LYEVE_SETUP_TOKEN`. Without it each pod prints its own token and setup works only on that pod |

The engine logs at boot whether sessions are shared: look for the `refresh token store` line.
The Redis cache and the Redis rate limit are included free on every install.
[Scaling](../scaling/index.md) lists the rest of what is shared and what is per replica.

## Configuration and secrets

Non-secret settings go in a ConfigMap and secrets in a Secret. The engine takes both with
`envFrom`, so every key becomes an environment variable. All settings are on
[Configuration](../reference/configuration.md).

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lyeve
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: lyeve-config
  namespace: lyeve
data:
  RATE_LIMIT_RPS: "100"
  RATE_LIMIT_BACKEND: "redis"
  CACHE_DRIVER: "redis"
  SECURE_COOKIE: "true"
  JWT_EXPIRY_SECS: "900"
  DATABASE_MAX_CONNECTIONS: "20"
  CORS_ORIGINS: "https://admin.example.com"     # cluster-specific
  LYEVE_CONSOLE_URL: "https://admin.example.com" # cluster-specific: the links the engine emails open here
  TRUSTED_PROXIES: "10.0.0.0/8"                 # cluster-specific: your pod network CIDR
  STORAGE_S3_BUCKET: "lyeve-media"              # cluster-specific
  STORAGE_S3_REGION: "us-east-1"                # cluster-specific
  LYEVE_PLUGINS: ""                             # empty starts every licensed feature
---
apiVersion: v1
kind: Secret
metadata:
  name: lyeve-secrets
  namespace: lyeve
type: Opaque
stringData:
  DATABASE_URL: "postgres://lyeve:<password>@db-host:5432/lyeve?sslmode=require"  # cluster-specific
  REDIS_URL: "redis://redis.lyeve.svc:6379/0"   # cluster-specific
  JWT_SECRET: "<openssl rand -hex 32>"
  ENCRYPTION_KEY: "<a different openssl rand -hex 32>"
  LYEVE_AUDIT_HMAC_KEY: "<openssl rand -hex 32>"
  ADMIN_CONSOLE_KEY: "<openssl rand -hex 32>"
  LYEVE_SETUP_TOKEN: "<openssl rand -hex 24>"
  STORAGE_S3_KEY: "<access key id>"             # cluster-specific
  STORAGE_S3_SECRET: "<secret access key>"      # cluster-specific
  LYEVE_LICENSE_KEY: ""                         # empty runs the free features only
```

`APP_ENV` is absent because it defaults to `production`, which refuses a weak configuration.
[Configuration](../reference/configuration.md#what-production-refuses) lists every check.

`TRUSTED_PROXIES` names the addresses your Ingress controller connects from. The engine then
reads the client address from `X-Forwarded-For`. Without it every request counts as the
controller's, and the per-IP limits apply to all users together.

`ADMIN_CONSOLE_KEY` is shared with the admin console Deployment below. The console signs each call with
the browser's address, so sign-in limits count each user rather than the console pods.

Do not commit a real Secret. Render it from a secret manager the cluster trusts, such as
external-secrets, Sealed Secrets or SOPS. For MySQL or SQL Server, use the matching URL. The
engine reads the database type from it.

## The signing key

The engine signs tokens with an Ed25519 key that it creates on first boot at `JWT_KEY_PATH`
(default `/var/lib/lyeve/jwt_key.json`). On a pod's own filesystem that key is lost on every
restart, and each replica creates a different one, so a token from one pod fails on another.
Give every pod the same key from a Secret.

Generate the key file once with OpenSSL:

```bash
openssl genpkey -algorithm ed25519 -out ed25519.pem
seed=$(openssl pkey -in ed25519.pem -outform DER | tail -c 32 | base64)
pub=$(openssl pkey -in ed25519.pem -pubout -outform DER | tail -c 32 | base64)
kid=$(openssl pkey -in ed25519.pem -pubout -outform DER | tail -c 32 \
  | openssl dgst -sha256 -binary | head -c 8 | base64 | tr '+/' '-_' | tr -d '=')
printf '{"seed":"%s","public_key":"%s","kid":"%s","created_at":"%s"}\n' \
  "$seed" "$pub" "$kid" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" > jwt_key.json

kubectl -n lyeve create secret generic lyeve-jwt-key --from-file=jwt_key.json
```

Keep `ed25519.pem` and `jwt_key.json` in your secret store, then delete the local copies. A new
key ends every session.

In production the engine refuses a key file that group or others can read. A Secret volume is
group-readable once the pod sets `fsGroup`, so the Deployment below copies the key into an
`emptyDir` with mode `0600` before the engine starts.

## Engine Deployment and Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lyeve-core
  namespace: lyeve
spec:
  replicas: 2
  strategy: { type: RollingUpdate }
  selector:
    matchLabels: { app: lyeve-core }
  template:
    metadata:
      labels: { app: lyeve-core }
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        runAsGroup: 65532
        fsGroup: 65532
        seccompProfile: { type: RuntimeDefault }
      # The engine waits PRESTOP_DRAIN_SECS (5) and then lets requests finish
      # for up to GRACEFUL_SHUTDOWN_SECS (60). Give it that long.
      terminationGracePeriodSeconds: 70
      initContainers:
        - name: signing-key
          image: busybox:1.36
          command: ["sh", "-c", "cp /key/jwt_key.json /state/jwt_key.json && chmod 0600 /state/jwt_key.json"]
          volumeMounts:
            - { name: jwt-key, mountPath: /key, readOnly: true }
            - { name: state, mountPath: /state }
      containers:
        - name: lyeve-core
          image: ghcr.io/lyeve-labs/lyeve-core:0.50.4   # cluster-specific: pin a release
          ports:
            - { name: admin, containerPort: 3001 }
            - { name: api, containerPort: 3002 }
          envFrom:
            - configMapRef: { name: lyeve-config }
            - secretRef: { name: lyeve-secrets }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - { name: state, mountPath: /var/lib/lyeve }
            - { name: tmp, mountPath: /tmp }
          startupProbe:
            httpGet: { path: /startup, port: admin }
            periodSeconds: 2
            failureThreshold: 60
          livenessProbe:
            httpGet: { path: /healthz, port: admin }
            periodSeconds: 15
          readinessProbe:
            httpGet: { path: /readyz, port: admin }
            periodSeconds: 10
          resources:
            requests: { cpu: 250m, memory: 512Mi }
            limits: { memory: 1Gi }
      volumes:
        - name: jwt-key
          secret: { secretName: lyeve-jwt-key, defaultMode: 0440 }
        - name: state
          emptyDir: {}
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: lyeve-core
  namespace: lyeve
spec:
  selector: { app: lyeve-core }
  ports:
    - { name: admin, port: 3001, targetPort: admin }
    - { name: api, port: 3002, targetPort: api }
```

Every pod applies pending migrations when it starts, under a database lock, so pods that start
together apply each migration once. A pod waits up to three minutes for another pod's
migrations before it gives up. [Scaling](../scaling/index.md) covers schema changes and caches
across replicas.

### Probes

Both listeners serve three probes at the root, with no authentication and over plain HTTP even
with `SECURE_COOKIE=true`:

| Path | Probe |
|---|---|
| `/startup` | Startup. `200` once every check has passed once |
| `/healthz` | Liveness. `503` when the database does not answer a ping |
| `/readyz` | Readiness. `503` while starting, while shutting down, when the database is unreachable, or when less than 100 MiB is free on the uploads path |

On shutdown `/readyz` turns `503` for `PRESTOP_DRAIN_SECS` first, so the Service stops sending
traffic before the pod stops accepting it. Do not probe `/api/admin/health` or `/api/v1/health`: they need
authentication and answer `401`. With `TLS_CERT_FILE` set the listeners speak TLS, so add
`scheme: HTTPS` to each probe.

### gRPC ports

The image also exposes **3003** and **3004** for the [gRPC API](../features/grpc-api.md), which
needs a license with the `grpc` feature. The listener binds loopback until told otherwise. To
open it, add two keys to the ConfigMap and two ports to the container and the Service:

```yaml
# lyeve-config ConfigMap, under data:
GRPC_ADDR: "0.0.0.0:3003"
GRPC_HEALTH_ADDR: "0.0.0.0:3004"
```

```yaml
# lyeve-core container, under ports:
- { name: grpc, containerPort: 3003 }
- { name: grpc-health, containerPort: 3004 }
```

```yaml
# lyeve-core Service, under spec.ports:
- { name: grpc, port: 3003, targetPort: grpc }
- { name: grpc-health, port: 3004, targetPort: grpc-health }
```

The gRPC listener speaks plaintext HTTP/2. Terminate TLS at the Ingress, with an `h2c` backend
on most controllers, or keep it inside the cluster. Port 3004 is for probes, not a public route.

## Admin console Deployment and Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lyeve-admin
  namespace: lyeve
spec:
  replicas: 1
  selector:
    matchLabels: { app: lyeve-admin }
  template:
    metadata:
      labels: { app: lyeve-admin }
    spec:
      containers:
        - name: lyeve-admin
          image: ghcr.io/lyeve-labs/lyeve-admin:0.17.0   # cluster-specific: pin a release
          ports:
            - { name: http, containerPort: 3002 }
          env:
            - { name: ORIGIN, value: "https://admin.example.com" }                 # cluster-specific
            # Must route /api/admin to port 3001 and the rest of /api to 3002.
            # The Ingress host does.
            - { name: CORE_INTERNAL_URL, value: "https://admin.example.com" }      # cluster-specific
            - { name: CORE_API_INTERNAL_URL, value: "https://admin.example.com" }  # cluster-specific
            - { name: ADDRESS_HEADER, value: "X-Forwarded-For" }
            - { name: XFF_DEPTH, value: "1" }
            - name: ADMIN_CONSOLE_KEY
              valueFrom:
                secretKeyRef: { name: lyeve-secrets, key: ADMIN_CONSOLE_KEY }
          readinessProbe:
            httpGet: { path: /, port: http }
            periodSeconds: 10
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits: { memory: 256Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: lyeve-admin
  namespace: lyeve
spec:
  selector: { app: lyeve-admin }
  ports:
    - { name: http, port: 80, targetPort: http }
```

The console's server sends every `/api/*` call to `CORE_INTERNAL_URL`. A base of
`http://lyeve-core.lyeve.svc:3001` would send Content API calls to the Admin API listener, which
answers them with `404`, so point it at the Ingress host, which routes by path.
`CORE_API_INTERNAL_URL` is where the API reference page sends the calls it runs under `/api/v1`.
The console refuses to start without it, so it names the same Ingress host.

The console runs as one replica. It renews a session by trading the session's refresh token
with the engine, and the engine ends the session when a token it already traded comes back.
Two replicas renewing one session at the same moment send that token twice, so the user is
signed out. To run more than one, give the admin host cookie affinity at the Ingress, so each
browser stays on one replica. On ingress-nginx that is the
`nginx.ingress.kubernetes.io/affinity: cookie` annotation.

## Ingress

The browser loads the console and also calls the engine on the console's own origin, so the
admin host routes by path. Kubernetes matches the longest prefix first.

| Host | Path | Service |
|---|---|---|
| admin | `/api/admin` | `lyeve-core` 3001 |
| admin | `/api` | `lyeve-core` 3002 |
| admin | `/.well-known` | `lyeve-core` 3002 |
| admin | `/` | `lyeve-admin` 80 |
| api | `/` | `lyeve-core` 3002 |

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lyeve
  namespace: lyeve
  annotations: {}                                     # cluster-specific: cert-manager issuer, etc.
spec:
  ingressClassName: nginx                             # cluster-specific
  tls:
    - hosts: ["admin.example.com", "api.example.com"] # cluster-specific
      secretName: lyeve-tls                           # cluster-specific
  rules:
    - host: admin.example.com                         # cluster-specific
      http:
        paths:
          - path: /api/admin
            pathType: Prefix
            backend: { service: { name: lyeve-core, port: { number: 3001 } } }
          - path: /api
            pathType: Prefix
            backend: { service: { name: lyeve-core, port: { number: 3002 } } }
          - path: /.well-known
            pathType: Prefix
            backend: { service: { name: lyeve-core, port: { number: 3002 } } }
          - path: /
            pathType: Prefix
            backend: { service: { name: lyeve-admin, port: { number: 80 } } }
    - host: api.example.com                           # cluster-specific
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: lyeve-core, port: { number: 3002 } } }
```

TLS is terminated at the Ingress, which sets `X-Forwarded-Proto: https` for the engine. With
`SECURE_COOKIE=true` the engine redirects any request that does not carry it, so TLS is not
optional here.

## Verify

```bash
kubectl -n lyeve rollout status deploy/lyeve-core
kubectl -n lyeve logs deploy/lyeve-core | grep 'refresh token store'
curl https://api.example.com/.well-known/jwks.json
curl https://admin.example.com/api/admin/setup     # {"setup_required":true,"token_source":"env"}
```

Every pod serves the same JWKS key. `token_source` is `env` because `LYEVE_SETUP_TOKEN` is set.
Open `https://admin.example.com`, enter that token on the setup page, and create the first
administrator.

## One replica

A single engine pod needs no Redis and no bucket. Set `replicas: 1`, remove `CACHE_DRIVER`,
`RATE_LIMIT_BACKEND`, `REDIS_URL` and the `STORAGE_S3_*` keys, and replace the `state` emptyDir,
the init container and the key Secret with persistent volumes:

```yaml
# lyeve-core pod template, under spec:
containers:
  - name: lyeve-core
    volumeMounts:
      - { name: state, mountPath: /var/lib/lyeve }
      - { name: uploads, mountPath: /app/uploads }
      - { name: tmp, mountPath: /tmp }
volumes:
  - name: state
    persistentVolumeClaim: { claimName: lyeve-state }
  - name: uploads
    persistentVolumeClaim: { claimName: lyeve-uploads }
  - name: tmp
    emptyDir: {}
```

With two `ReadWriteOnce` claims named `lyeve-state` (1Gi) and `lyeve-uploads` (10Gi), the engine
creates its key on the state volume and keeps media on the uploads volume. Use
`strategy: { type: Recreate }`, because a `ReadWriteOnce` claim cannot attach to the new pod
while the old one holds it.

## Before you go live

- Pin both image tags to a release. `latest` makes rollouts and rollbacks unpredictable.
- Check the boot log for the features that started. A feature starts when it is free or your
  license includes it, and when `LYEVE_PLUGINS` is empty or names it. See
  [Licensing and tiers](../concepts/licensing-and-tiers.md).
- Work through the [production checklist](production-checklist.md).
