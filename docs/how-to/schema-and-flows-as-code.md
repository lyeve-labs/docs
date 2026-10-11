---
title: Keep the Content Model and Flows in Git
description: "Pull the content model and the flows into your repository, review changes in pull requests, gate them in CI, and promote both on release."
sidebar:
  order: 12
---

> **Included free** on every install. A flow that uses a custom path, another protocol or an outbound node requires a license with the `flow-pro` feature. See [pricing](https://lyeve.com/pricing).

Clicking the same change into staging and then into production is how the two
drift apart. This guide puts the content model and the flows in your
repository instead: the model as one bundle file, each flow as its own file.
A change becomes a pull request, CI shows what it would do to the database,
and on release you bring the instance up to date from the files. Everything
here is a call to the Admin API, so it runs from a terminal or a CI job with
`curl` and `jq`.

## Before you start

- A running instance whose Admin API you can reach, and a signed-in super
  admin. Planning and applying a model change is served only to a super admin
  with a signed-in session, never to an admin token or an API key.
- `curl` and `jq`.
- The flow steps run on the free tier. A flow with a custom path, a protocol
  other than REST or a node that calls out needs `flow-pro`, and importing one
  without it answers `402`.

## 1. Sign in

Set the instance once, and sign in for a session token:

```bash
export ADMIN_URL=https://staging.example.com

export TOKEN=$(curl -s "$ADMIN_URL/api/admin/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "<password>"}' | jq -r .token)
```

`TOKEN` now holds a session token. If it says `null`, the account signs in
with a second factor, and the answer carried a challenge instead. Finish it as
[multi-factor authentication](../features/multi-factor-auth.md) shows. On an instance with
several tenants, add `-H "X-Tenant-ID: acme"` to each call to act in that
tenant. For the export and the flow steps alone, an
[admin token](admin-tokens.md) with `schemas:read`, `flows:read` and
`flows:write` works too, in the tenant it was issued for.

## 2. Pull the model into the repository

```bash
curl -sf "$ADMIN_URL/api/admin/schemas/export" \
  -H "Authorization: Bearer $TOKEN" > schemas.yaml
```

`schemas.yaml` holds every content type of the tenant, in dependency order,
with every field the instance stores, including the `id` and relation key
fields it adds itself. An excerpt:

```yaml
schemas:
  - display_name: Article
    fields:
      - field_type: uid
        indexed: true
        name: id
        required: true
        system: true
        unique: true
      - field_type: text
        indexed: true
        name: title
        required: true
        unique: false
      - field_type: relation
        indexed: false
        name: author
        relation_to: author
        relation_type: belongs_to
        required: false
        unique: false
      - field_type: uid
        indexed: true
        name: author_id
        required: false
        system: true
        unique: false
    name: article
    with_created_at: true
    with_draft_publish: true
source: lyeve/staging-1
version: 1
```

Commit it. From now on a model change starts as an edit to this file. Add
`?format=json` for JSON. [Moving content types between projects](../reference/content-type-portability.md)
describes the format.

## 3. Change the model and read the plan

Edit `schemas.yaml`: add a field, mark one required, add a content type. Then
ask the instance what importing the file would do. Without `apply=true` the
import is a dry run and writes nothing:

```bash
curl -sf -X POST "$ADMIN_URL/api/admin/schemas/import" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/yaml" \
  --data-binary @schemas.yaml > plan.json

jq -r '.plan.schemas[] | "\(.action)\t\(.name)", (.ddl[]? | "    " + .)' plan.json
```

```text
unchanged	author
update	article
    ALTER TABLE "_article" ADD COLUMN IF NOT EXISTS "summary" TEXT
```

Each content type has an `action`, `create`, `update` or `unchanged`, and the
`ddl` it would run, written for your database. A field you removed shows up as
`DROP COLUMN`. A content type that the file leaves out is never touched, so a
file that happens not to list one is not an instruction to delete it.

A content type that points at one that is neither in the file nor on the
instance is listed with `missing`, and applying the file answers `409` with
`import would fail: [<names>] reference schemas that are neither in the bundle nor in the target`.

## 4. Gate the change in CI

Run the dry run against staging on every pull request that touches the file,
and fail the job when the plan has something missing or removes a column:

```yaml
- name: Plan the content model
  env:
    ADMIN_URL: ${{ vars.STAGING_ADMIN_URL }}
    TOKEN: ${{ secrets.STAGING_SESSION_TOKEN }}
  run: |
    curl -sf -X POST "$ADMIN_URL/api/admin/schemas/import" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/yaml" \
      --data-binary @schemas.yaml > plan.json
    if jq -e '[.plan.schemas[] | select((.missing // []) | length > 0)] | length > 0' plan.json >/dev/null; then
      echo "::error::the model references a content type the instance does not have"; exit 1
    fi
    if jq -e '[.plan.schemas[].ddl[]? | select(test("DROP (COLUMN|TABLE)"; "i"))] | length > 0' plan.json >/dev/null; then
      echo "::error::the model change removes a column; review it before release"; exit 1
    fi
```

The token has to be a super admin's session token, signed in from the job with
credentials kept in your secret store. Keep `plan.json` as a build artifact so
the reviewer reads the plan the gate read. A field whose type changes shows up
as `ALTER COLUMN ... TYPE`. Read those in review, because a narrower type can
change or lose existing values.

## 5. Apply on release

```bash
curl -sf -X POST "$ADMIN_URL/api/admin/schemas/import?apply=true" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/yaml" \
  --data-binary @schemas.yaml
```

The answer is the same plan with `"applied": true`. Every statement runs
except column removals, which wait as pending so their data stays until a super
admin applies them. See [Apply removals](../features/schema-changes.md#apply-removals).

Two content types that point at each other cannot be created by one import:
the apply answers `409`. Import the file once with one of the two relation
fields left out, then import the full file.

## 6. Pull the flows

Export every flow to `flows/<slug>.yaml`:

```bash
mkdir -p flows
curl -sf "$ADMIN_URL/api/admin/flows?limit=200" \
  -H "Authorization: Bearer $TOKEN" |
  jq -r '.data[] | "\(.id) \(.slug)"' |
  while read -r id slug; do
    curl -sf "$ADMIN_URL/api/admin/flows/$id/export?format=yaml" \
      -H "Authorization: Bearer $TOKEN" > "flows/$slug.yaml"
  done
```

The list returns at most 200 flows a call, so page with `offset` past that.
Each file is the flow's current draft in the
[flow definition format](../reference/flow-definition.md), canvas positions
included. A datasource is named, never included. A literal value under a
setting whose name marks a credential, such as a webhook `secret` or an
`Authorization` header, is left out of the file, while a `{{ vars.<key> }}`
reference stays. Keep secrets in variables and the files are safe to commit.
A change to a flow is now a diff of
nodes, edges and settings that a reviewer can read.

## 7. Push a flow

Import a file as the new draft of the flow with that slug:

```bash
curl -s -X POST "$ADMIN_URL/api/admin/flows/import?mode=replace&slug=orders-with-customer" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@flows/orders-with-customer.yaml"
```

The answer is `200` with the flow and `unresolved_datasources`, the datasource
names the file uses that this tenant does not have. The published version
keeps running until you publish. A slug the instance does not have answers
`404`. Create that flow with `mode=create` instead, which answers `201`.

## 8. Publish

```bash
curl -s -X POST "$ADMIN_URL/api/admin/flows/$FLOW_ID/publish" \
  -H "Authorization: Bearer $TOKEN"
```

`FLOW_ID` is the `id` from the import answer. Publishing stores the draft as
the next version and makes it the one that runs. The previous version stays
listed, and `POST /api/admin/flows/{id}/rollback` with `{"version": n}` brings
it back.

## Promote on release day

From the repository that holds both, against production:

1. Run the dry run and read the plan.
2. Apply it once the plan holds only what you meant.
3. Push each flow with `mode=replace`, then publish it.

The model goes first, because a flow that reads a field the content type does
not have yet fails on its first run.

## Next

- [Promote configuration](../features/promote-configuration.md): move the
  model, every flow, the access rules and the webhooks in one bundle, with a
  license that carries `config-sync`.
- [Moving content types between projects](../reference/content-type-portability.md):
  the bundle format, and importing from other systems.
- [Schema changes](../features/schema-changes.md): preview, rename and apply
  removals.
- [Flow templates](flow-templates.md): starter flows to export and keep.
- [Admin tokens](admin-tokens.md): a credential for the export and flow steps.
