---
title: Source code
description: "Check which parts of LyEve are MIT-licensed, which are commercial, and which repositories you can read today."
sidebar:
  order: 2
---

Two parts of LyEve are MIT-licensed, and each image carries the MIT `LICENSE`
for the code in it:

- **The engine.** First-run setup, sign-in, users and roles, and the
  PostgreSQL, MySQL and SQL Server support. It answers the Admin API and the
  Content API. On its own it serves no content, because content types come
  from a feature.
- **The admin console.**

Three parts are commercial, and their source is not public:

- **License verification.** The server in the engine image includes it, under
  the [EULA](https://lyeve.com/legal/eula/). It holds no secret, because
  verification is an offline Ed25519 signature check against a public key.
- **Every feature beyond the engine, including the ones included free.** They
  are commercial software under the [EULA](https://lyeve.com/legal/eula/). Free
  means free of charge, not open source. Content types, entries, draft and
  publish, revisions and scheduled publishing come from two free features,
  [schema changes](../features/schema-changes.md) and
  [content lifecycle](../features/content-lifecycle.md). Access rules are a
  feature too.
- **The visual flow and schema editors in the admin image.** They are under
  the [EULA](https://lyeve.com/legal/eula/), and the admin image ships a summary
  of it beside the MIT license as `LICENSE-COMMERCIAL.md`.

## Check the licenses in the images

Each image says what it contains in its `org.opencontainers.image.licenses`
label:

```bash
docker inspect --format '{{ index .Config.Labels "org.opencontainers.image.licenses" }}' \
  ghcr.io/lyeve-labs/lyeve-core:latest
```

| Image | Label | Why |
|---|---|---|
| `ghcr.io/lyeve-labs/lyeve-core` | `LicenseRef-LyEve-Commercial` | The server in it also contains license verification and the features. |
| `ghcr.io/lyeve-labs/lyeve-admin` | `MIT AND LicenseRef-LyEve-Commercial` | The console is MIT, and the visual editors are commercial. |

The engine image keeps its `LICENSE` and `THIRD_PARTY_NOTICES.md` in
`/usr/share/doc/lyeve/`. Copy them out without starting the image:

```bash
id=$(docker create ghcr.io/lyeve-labs/lyeve-core:latest)
docker cp "$id":/usr/share/doc/lyeve/LICENSE ./LICENSE
docker cp "$id":/usr/share/doc/lyeve/THIRD_PARTY_NOTICES.md ./THIRD_PARTY_NOTICES.md
docker rm "$id" >/dev/null
```

## Repositories you can read

Fourteen repositories are public on
[github.com/lyeve-labs](https://github.com/lyeve-labs), all MIT-licensed. The
engine, the admin console and this documentation are three of them:

| Repository | What it is |
|---|---|
| [`lyeve-core`](https://github.com/lyeve-labs/lyeve-core) | The engine |
| [`lyeve-admin`](https://github.com/lyeve-labs/lyeve-admin) | The admin console, without the visual editors |
| [`lyeve-examples`](https://github.com/lyeve-labs/lyeve-examples) | Runnable example applications |
| [`benchmarks`](https://github.com/lyeve-labs/benchmarks) | The performance benchmarks, to rerun on your own hardware |
| [`docs`](https://github.com/lyeve-labs/docs) | This documentation. Open an issue or a pull request to fix a page |

The other nine are the client libraries and the UI kit, each published on npm:

| Repository | Published on npm as |
|---|---|
| `client` | `@lyeve-labs/client` |
| `client-rest` | `@lyeve-labs/client-rest` |
| `client-graphql` | `@lyeve-labs/client-graphql` |
| `client-grpc` | `@lyeve-labs/client-grpc` |
| `client-realtime` | `@lyeve-labs/client-realtime` |
| `client-react` | `@lyeve-labs/client-react` |
| `client-svelte` | `@lyeve-labs/client-svelte` |
| `client-vue` | `@lyeve-labs/client-vue` |
| `ui-kit` | `@lyeve-labs/ui-kit` |

## What this means in practice

- **The product you run is the image** `ghcr.io/lyeve-labs/lyeve-core`, which
  carries the engine, license verification and every feature. A paid feature
  stays off until a license includes it.
  [Verify Offline](verify-offline.md) shows how to check the binary and that,
  given a signed token, it never calls home.
- **The MIT license covers the engine and admin code**, including the right to
  modify and redistribute it. Read it, fork it and file bugs in
  [lyeve-core](https://github.com/lyeve-labs/lyeve-core/issues) and
  [lyeve-admin](https://github.com/lyeve-labs/lyeve-admin/issues). Support
  requests go through [lyeve.com/contact](https://lyeve.com/contact/).
- **The client libraries are fully open.** You can read, fork and build every
  library your application links against.

## Why this shape

MIT on the part that handles your data is what lets you keep running it, and
change it, whatever happens to us. The features beyond the engine are where the
product's commercial value is, and license verification is what enforces the
plan you bought. Keeping both commercial is what makes a self-hosted license
mean anything.
