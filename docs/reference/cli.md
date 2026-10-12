---
title: Server binary
description: "Run the server binary, use its built-in health check, and know what happens to migrations when it starts."
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
that start together wait for each other, so only one applies them. There is no separate
migration command.

## Check health from inside the container

```bash
lyeve healthcheck
```

This requests `/healthz` on `127.0.0.1` at port `PORT` (default `3002`), over HTTPS when
`TLS_CERT_FILE` is set. It exits `0` on a `200` and `1` otherwise, without starting a server.
The image runs it as its container health check. If you move the Content API off port 3002,
set `PORT` to match.

`healthcheck` is the only argument the binary reads. Any other argument is ignored and the
server starts.

## Related

- [Docker images](../deploy/docker.md): ports, volumes and tags.
- [Configuration](configuration.md): every setting the server reads.
