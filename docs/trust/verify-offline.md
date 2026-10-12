---
title: Verify offline
description: "Confirm on your own machine that, given a signed license token, the server makes no outbound call to verify a license or gate a feature."
sidebar:
  order: 1
---

License verification in LyEve is a local Ed25519 signature check. Set
`LYEVE_LICENSE_KEY` to the signed token from your account and the server never
contacts a license server, at startup or afterwards, and there is no sidecar or
agent. Every claim below is something you can check on your own Linux machine.

The other value `LYEVE_LICENSE_KEY` accepts, an opaque license key
(`lyeve_live_...`), is not offline. The server sends that key and a random
install id to the license server (`LYEVE_LICENSE_SERVER_URL`, default
`https://api.lyeve.com`) to receive a token, verifies the token locally and
caches it. It contacts the server again only when the cached token is within
seven days of expiry, or when it holds no active token, and a running server
checks for that every six hours. The checks on this page assume the token. With the key
they show that one https connection. For an air-gapped install, get a token
from the customer portal and set that.

## Get the binary

The server you run is built by LyEve and includes license verification, whose
source is not public (see [Source code](source-code.md)). So the checks start
from the published image:

```bash
# Resolve the release tag to its digest, then pin the digest.
# A digest names exactly one image, and a tag can be moved.
docker buildx imagetools inspect ghcr.io/lyeve-labs/lyeve-core:<version> | grep -m1 Digest
IMAGE=ghcr.io/lyeve-labs/lyeve-core@sha256:<that digest>
id=$(docker create "$IMAGE")
docker cp "$id":/app/lyeve ./lyeve
docker rm "$id" >/dev/null
```

The checks below run `./lyeve` directly on the host with the same environment
you give the container, including `DATABASE_URL`. They block the network, so
the database has to be reachable without it, or the server cannot start for
reasons that have nothing to do with the license.

## Quick check: run without a network

```bash
# A new network namespace with only a loopback interface
unshare -n --map-root-user sh -c 'ip link set lo up && ./lyeve'
```

The namespace has its own loopback, so a database listening on the host's
`127.0.0.1` is out of reach from inside it. Use a database reached through a
Unix socket, or use the firewall check below, which keeps the host's loopback.

With `LYEVE_LICENSE_KEY` set to your token, the log shows `license: active`
and the licensed features start. With an opaque key and no cached token, the key
cannot be exchanged, so `GET /api/admin/entitlements` reports a `license_error`
saying the license server could not be reached, and the install runs the free
tier.

## Deep check: system call trace

```bash
# Trace every network-related system call during startup
strace -f -e trace=connect,sendto,sendmsg,recvfrom,recvmsg \
  ./lyeve 2>&1 | grep -v "ENOTCONN\|EAGAIN"
```

What must never appear is a `connect()` to an outside address or to port 53, the
port DNS lookups use. Connections to `127.0.0.1` (the database, a local Redis)
are expected.

## DNS check

```bash
strace -f -e trace=connect -o ./connect.log ./lyeve &
sleep 10
grep -c 'htons(53)' ./connect.log
# Expected: 0
```

Stop the server when you are done.

strace writes a DNS lookup as a `connect` with `sin_port=htons(53)`, so a count
of zero means the server resolved no name while it started and verified the
license.

## Firewall check

Run this from a local console rather than over SSH, and as root:

```bash
# Allow loopback and replies to existing connections, drop every new outbound one
iptables -I OUTPUT 1 -j DROP
iptables -I OUTPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -I OUTPUT 1 -o lo -j ACCEPT

# If the server starts and licensed features run, nothing had to leave the machine
./lyeve

# Clean up
iptables -D OUTPUT -o lo -j ACCEPT
iptables -D OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -D OUTPUT -j DROP
```

On a host with IPv6, repeat the same rules with `ip6tables`.

## What offline covers

| Operation | How | Network |
|---|---|---|
| License verification | Ed25519 signature check against a built-in public key | No |
| Grace period after expiry | The token cached in `LYEVE_LICENSE_CACHE_DIR` and the local clock | No |
| Feature gating | The verified license, held in memory | No |
| Applying a new token | `POST /api/admin/license/renew` with the token, verified locally | No |
| License key exchange (opaque key only, never a token) | https POST of the key and install id to the license server | Yes |

The server mints no license of any kind. A license reaches it only as a signed
token, pasted or returned by the key exchange, which it verifies with a public
key and nothing else.

## What does use the network

These are features you configure, not license enforcement:

| Operation | When |
|---|---|
| Database connections | Always |
| Redis | When `REDIS_URL` or a Redis cache driver is configured |
| Sending email | When a mail provider is configured |
| Webhooks and flows calling URLs | When you create them |
| OAuth, SAML and SCIM | When an identity provider is configured |
| Message brokers (NATS, Kafka, RabbitMQ) | When configured |
| S3-compatible storage | When an S3 storage provider is configured |
| The AI feature's model provider | When you add a provider |
| Metrics and trace exporters | When an exporter endpoint is configured |

None of these is used for license enforcement. Leave them unconfigured, set a
signed token, and the server still starts, verifies the license and gates
features with no network.

## Which LyEve hosts the binary names

```bash
strings ./lyeve | grep -o "[a-z0-9.-]*lyeve\.com" | sort -u
```

This prints `api.lyeve.com`, the license server an opaque key is exchanged
with, plus the hosts of links the server hands out: `app.lyeve.com` (pricing,
the customer portal and support, shown in the admin console) and
`docs.lyeve.com` (documentation links). Only `api.lyeve.com` is ever contacted,
and only for an opaque key. The traces above
are what prove that.

The proof that the built-in public key is the right one is the license itself:
a token issued for your install verifies against it offline, and
`GET /api/admin/entitlements` shows what it opened. A token that does not verify
is refused. At startup the log says
`license: verification failed, checking grace cache`, and the renew route
answers `400`.

## Checklist

- With a signed token, the server starts and licensed features run with no
  outbound network.
- strace shows no `connect()` to an outside address.
- strace shows no connection to port 53.
- The only lyeve.com host ever contacted is `api.lyeve.com`, and only for an
  opaque key.
- After the token expires, the install keeps its paid features through the
  seven-day grace period.
