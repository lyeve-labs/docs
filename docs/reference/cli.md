---
title: Server binary
description: "Run the server binary, use its built-in health check, know what happens to migrations when it starts, and repair a lost migration record."
sidebar:
  order: 3
---

The engine image carries one binary, `lyeve`, at `/app/lyeve`, and runs it as its entrypoint.
There is no separate command-line client to install. Everything else is done in the admin
console or over the HTTP APIs, which [API endpoints](api-endpoints.md) lists.

## Run the server

```bash
lyeve
```

It serves the Admin API on `ADMIN_LISTEN_ADDR` and the Content API on `API_LISTEN_ADDR`, which
the image sets to `0.0.0.0:3001` and `0.0.0.0:3002`. Settings come from environment variables
and an optional `lyeve.yaml`, as [Configuration](configuration.md) describes.

Every pending database migration runs at start, before the server answers a request. Replicas
that start together wait for each other, so only one applies them. There is no command to run
migrations by hand. The one migration command repairs a lost record, as described below.

## Check health from inside the container

```bash
lyeve healthcheck
```

This requests `/healthz` on `127.0.0.1` at port `PORT` (default `3002`), over HTTPS when
`TLS_CERT_FILE` is set. It exits `0` on a `200` and `1` otherwise, without starting a server.
The image runs it as its container health check. If you move the Content API off port 3002,
set `PORT` to match.

## Print the version

```bash
lyeve --version
```

This prints the version, commit and build date and exits `0`, without opening the database or
binding a port. `-version` and `version` do the same.

## Record a migration the database already holds

A start can stop on a feature's migration with an error that says the schema already holds what
the migration creates, and that this install may have lost its record of it. That happens after
a hand repair or a partial restore that kept the schema but not the record. The error names the
command to run:

```bash
lyeve migrate mark-applied --plugin <plugin> --version <version>
```

Take `<plugin>` and `<version>` from the error. First check in the database that the change the
migration makes is really there, because the command records the version as applied without
running its script. Then run it from a shell on the engine's host, or with `docker exec` in its
container, so it reads the same `DATABASE_URL` as the engine. It has no HTTP route, so a console
session cannot run it. On success it prints the bookkeeping table it wrote to, and the next start
goes on past that migration.

It refuses, and exits `1`, when:

- the feature is not part of this binary, or does not declare its migrations
- the feature ships no migration with that version for this database
- that version is already recorded as applied
- an engine is applying migrations on this database at that moment. Run it again once that
  engine has finished starting.

## Any other argument

Any other argument prints the usage line and exits `2`, so a mistyped flag never starts a second
server beside a running one.

## Related

- [Docker images](../deploy/docker.md): ports, volumes and tags.
- [Configuration](configuration.md): every setting the server reads.
