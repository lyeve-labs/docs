---
title: Data residency
description: "Define data regions, assign each tenant to one, refuse writes that reach the wrong region, and report where every tenant's data lives."
sidebar:
  order: 22
---

> **Requires** a license with the `data-residency` feature. See [pricing](https://lyeve.com/pricing).

Data residency records where each tenant's data must live and holds every
engine to it. You define the regions you run in, assign each
[tenant](tenants.md) to one, and start each engine with the region it serves. A
write from a tenant that belongs in another region is refused and told where to
go.

:::tip[Best for]
- **Contracts that keep data in one country.** Assign an EU customer to your EU
  region, and an engine in the US refuses that customer's writes.
- **A sign-up page that picks the closest region.** Two public routes list your
  regions and return the nearest one to a point, with no credentials.
- **Answering an auditor.** One report lists every region, every tenant's
  assignment and how many tenants have none.
- **Keeping a record of your recovery plan.** Note each tenant's replica region
  with its recovery targets, and planned moves between regions.
:::

## How enforcement works

When an engine starts with `INSTANCE_REGION` set, the Content API checks every
write (any method except `GET`, `HEAD` and `OPTIONS`):

| Tenant | Result |
|---|---|
| No region assigned | Writes anywhere. |
| Assigned to this engine's region | Writes as usual. |
| Assigned to another region | `421 Misdirected Request`. |
| Region cannot be read | `503` with `cannot determine the tenant's data region`. The write is never let through. |

```http
HTTP/1.1 421 Misdirected Request
X-CMS-Region: us-east-1
X-CMS-Region-Target: eu-west-1
X-CMS-Region-Reason: data-residency-enforced

{"error":"write routed to wrong region","target_region":"eu-west-1"}
```

Route the request to an engine in `X-CMS-Region-Target`. Reads are not checked,
and neither is the Admin API. Without `INSTANCE_REGION`, nothing is enforced.
With it, every Content API response carries `X-CMS-Region` with this engine's
region, so a load balancer can route on it.
Residency decides where a tenant's writes land. It does not look at where a
client connects from.

Uploads follow the same assignment. A tenant's files go only to storage
providers in its region, and an upload whose region cannot be read answers
`503` rather than land elsewhere. See
[object storage](object-storage.md#keep-a-tenants-files-in-its-region).

## Try it

This defines a region, assigns a tenant to it and reads the report. You need a
super admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md) shows how to get one.

1. Create a region:

   ```bash
   curl -X POST http://localhost:3001/api/admin/regions \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"slug": "eu-west-1", "display_name": "EU West (Ireland)", "provider": "aws", "coordinates": {"lat": 53.35, "long": -6.26}}'
   ```

   The answer is `201` with the region. Copy its `id` into `REGION`.

2. Find the tenant's id with `GET /api/admin/tenants` and copy it into
   `TENANT`. A tenant is assigned by its id, not its slug.

3. Assign the tenant:

   ```bash
   curl -X POST http://localhost:3001/api/admin/tenants/$TENANT/region \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d "{\"region_id\": \"$REGION\"}"
   ```

   ```json
   {
     "id": "909c7ca6-f9ed-4445-82bb-51f62b836bd7",
     "tenant_id": "3f64a521-1146-481c-aaf8-a3334f505aa3",
     "region_id": "76e7b309-80f4-4341-a958-2b15dacbbe44",
     "region_slug": "eu-west-1",
     "assigned_at": "2026-10-02T09:14:56Z"
   }
   ```

4. Read the report:

   ```bash
   curl http://localhost:3001/api/admin/residency/report \
     -H "Authorization: Bearer $TOKEN"
   ```

   It lists `regions`, every tenant's assignment under `tenant_regions`, and a
   `summary`:

   ```json
   {
     "total_regions": 1,
     "total_tenants": 1,
     "tenants_per_region": {"eu-west-1": 1},
     "replicating_count": 0,
     "unassigned_count": 0
   }
   ```

5. Ask for the nearest region the way a sign-up page would, with no credentials:

   ```bash
   curl -X POST http://localhost:3002/api/v1/regions/nearest \
     -H "Content-Type: application/json" \
     -d '{"lat": 48.85, "long": 2.35}'
   ```

   The answer is the region with `distance_km`, about `781` from Paris to
   Dublin.

6. Open **Settings > Data residency** in the admin console. The region and the
   assignment are there.

## Turn it on

1. Run with a license that includes `data-residency`. If you set
   `LYEVE_PLUGINS`, include `data-residency` in it. See
   [Licensing and tiers](../concepts/licensing-and-tiers.md).
2. On each engine, set `INSTANCE_REGION` to the slug of the region it serves.
3. Create regions and assign tenants, in the console or over the API.

Without the license the feature does not start, and its routes answer `404`.

## Manage regions

Regions need a super admin. `slug` and `display_name` are required, and
`provider` defaults to `aws`. `coordinates` lets the nearest-region lookup use
the region. `is_default` marks the default region, and marking one clears the
mark from the others. `PUT` also takes `enabled`. Only enabled regions are
listed publicly.

An admin can read and set their own tenant's region. A super admin can do it for
any tenant. `GET /api/admin/tenants/{id}/region` returns the assignment, or `404`
with `no region assigned`.

## Record replicas and planned moves

These record your disaster-recovery setup and planned moves between regions.
The engine stores them and shows them in the report. Copying the data is up to
you.

- `POST /api/admin/tenants/{id}/replication` takes `replica_region_id`,
  `target_rpo_secs` (default `3600`) and `target_rto_secs` (default `1800`),
  and answers `201` with a record in status `syncing`.
- `PUT /api/admin/tenants/{id}/replication/{rid}` updates `status`, one of
  `syncing`, `synced`, `failed` or `paused`, and the lag.
- `POST /api/admin/tenants/{id}/migrate` takes `to_region_id` and an optional
  `requested_by`, and answers `202` with a migration record in status
  `estimating`.

## Public region lookup

Two Content API routes need no credentials:

- `GET /api/v1/regions` lists enabled regions, paginated (`limit` default 50,
  `offset`). 30 requests a second per address, bursts of 50.
- `POST /api/v1/regions/nearest` takes `{"lat": ..., "long": ...}` and answers
  `{"region": {...}, "distance_km": ...}`. 10 requests a second per address,
  bursts of 20.

## Settings

| Variable | What it does | Default |
|---|---|---|
| `INSTANCE_REGION` | The region slug this engine serves. Set, it turns on write enforcement. | unset |

:::caution[Records only]
Replica and migration entries are records you keep, and they move no data.
:::

## Routes

<details>
<summary>Regions, assignments and records</summary>

| Method | Path | Who |
|---|---|---|
| `GET`, `POST` | `/api/admin/regions` | super admin |
| `GET`, `PUT`, `DELETE` | `/api/admin/regions/{id}` | super admin |
| `GET`, `POST` | `/api/admin/tenants/{id}/region` | admin of that tenant |
| `GET`, `POST` | `/api/admin/tenants/{id}/replication` | admin of that tenant |
| `PUT`, `DELETE` | `/api/admin/tenants/{id}/replication/{rid}` | admin of that tenant |
| `POST` | `/api/admin/tenants/{id}/migrate` | admin of that tenant |
| `GET` | `/api/admin/tenants/{id}/migrations` | admin of that tenant |
| `GET` | `/api/admin/tenants/{id}/migrations/{mid}` | admin of that tenant |
| `GET` | `/api/admin/residency/report` | super admin |
| `GET` | `/api/v1/regions` | anyone |
| `POST` | `/api/v1/regions/nearest` | anyone |

</details>

## Errors

| Status | Message | Cause |
|---|---|---|
| `421` | `write routed to wrong region` | The tenant belongs in another region. |
| `403` | `forbidden: super_admin required` | A tenant admin called a region or report route. |
| `403` | `forbidden: cannot access another tenant's data` | A tenant admin named another tenant. |

<details>
<summary>Every error</summary>

| Status | Message | Cause |
|---|---|---|
| `400` | `slug is required`, `display_name is required` | Region created without them. |
| `400` | `region_id is required`, `to_region_id is required` | Assignment or migration without a region. |
| `400` | `invalid status: must be one of syncing, synced, failed, paused` | Unknown replication status. |
| `404` | `region not found`, `no region assigned`, `replication not found`, `migration not found` | Wrong id. |
| `409` | `region with slug "<slug>" already exists` | Duplicate slug. |
| `503` | `cannot determine the tenant's data region` | The tenant's region could not be read during a write. |

</details>

## Related

- [Tenants](tenants.md): what a tenant is and how a request picks one.
- [PII masking](pii-masking.md): keep personal data out of responses and logs.
- [Data protection](../trust/data-protection.md): how residency fits a vendor
  review.
