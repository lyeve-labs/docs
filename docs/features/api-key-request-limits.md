---
title: API key request limits
description: "Cap how many requests an API key may make in a calendar month, so a key you hand to a partner or a script cannot run up your traffic."
sidebar:
  order: 5
capability_of: apikey
---

> **Requires** a license with the `apikey-pro` feature. See [pricing](https://lyeve.com/pricing).

A request limit caps how many requests one [API key](api-keys.md) may make in a
calendar month. Once the key reaches it, every further request answers `429`
until the next month starts or you raise the limit. Everything else about keys,
from creating them to reading their history, stays free on every install.

:::caution[Beta]
This capability is in beta. It works as this page describes, and its fields can
still change before it is marked stable.
:::

:::tip[Best for]
- **A partner integration with a budget.** Give the partner's key 100,000
  requests a month, so their traffic cannot grow past what you agreed.
- **A build job or script that could loop.** A bug that calls the API in a
  tight loop stops at the limit instead of running all month.
- **Trial access.** Hand a prospect a key good for a few thousand requests.
- **A key you suspect has leaked.** Lower its limit at once while you move the
  real client to a new key. Lowering a limit is free.
:::

:::note[Use something else when]
- You want to slow bursts per second or per minute.
  [Rate limiting](rate-limiting.md) does that.
- You want to cap a whole tenant rather than one key.
  [Tenant quotas](tenant-quotas.md) cap a tenant's requests, storage and
  bandwidth.
:::

## How it works

| Question | Answer |
|---|---|
| What is counted | Every request the key makes on the Content API, whatever it answers, except the ones refused at the limit. |
| When the count resets | At the start of each calendar month, in UTC. |
| What a key over its limit gets | `429` with `X-RateLimit-Exceeded: true` and `{"error": "monthly request limit (100000) exceeded"}`. |
| What `0` means | No limit. It is the default. |
| Who counts | [Usage and quotas](usage-and-quotas.md), which runs on every install. If the count cannot be read, the request goes through. |

## Try it

Run this on an install whose license carries `apikey-pro`. You need an admin
token in `TOKEN`. The [quickstart](../getting-started/quickstart.md) shows how
to get one, and creates the `post` content type used here.

1. Create a key that may make three requests this month:

   ```bash
   curl -X POST http://localhost:3001/api/admin/api-keys \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name": "partner-feed", "scopes": ["content:read"], "monthly_limit": 3}'
   ```

   The answer is `201` with `"monthly_limit": 3`. Copy `raw_key` into `KEY`
   and `id` into `KEY_ID`.

2. Call the Content API four times with it:

   ```bash
   for i in 1 2 3 4; do
     curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3002/api/v1/content/post \
       -H "X-API-Key: $KEY"
   done
   ```

   You see `200`, `200`, `200`, then `429`.

3. Read the key's use this month:

   ```bash
   curl http://localhost:3001/api/admin/usage/api-key/$KEY_ID \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   { "api_key_id": "bab86a15-...", "tenant_id": "default", "billing_period": "2026-10", "requests": 3, "bytes_in": 0, "bytes_out": 9 }
   ```

   The refused fourth call is not counted.

4. Clear the limit, and the key is served again:

   ```bash
   curl -X PATCH http://localhost:3001/api/admin/api-keys/$KEY_ID/monthly-limit \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"monthly_limit": 0}'
   ```

   The answer is the key with `"monthly_limit": 0`.

5. In the admin console, **Access > API keys** shows how much of its limit each
   key has used this month, and lets you change the limit one key at a time.

## Set or change a limit

Give a new key a limit with `monthly_limit` on `POST /api/admin/api-keys`, as in
step 1, or change the limit of an existing key:

```bash
curl -X PATCH http://localhost:3001/api/admin/api-keys/$KEY_ID/monthly-limit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"monthly_limit": 50000}'
```

Both routes take an `admin` or `super_admin` with a signed-in session. An admin
token or another API key is refused.

| Write | Needs `apikey-pro` |
|---|---|
| Create a key with a `monthly_limit` above `0` | Yes |
| Give a key with no limit its first one | Yes |
| Raise a key's limit | Yes |
| Lower a key's limit | No |
| Clear a limit with `0` | No |
| Send back the value the key already holds | No |

## If the license lapses

Every limit you set keeps being enforced, because it caps what a client can
spend and a lapse must not lift it. Lowering and clearing a limit stay free, so
you can still tighten a key you worry about. Setting a first limit or raising
one needs `apikey-pro` again.

## Errors

| Status | Message | Cause |
|---|---|---|
| `400` | `validation failed on field "monthly_limit"` | The limit is below `0`. |
| `402` | `payment_required`, naming `feature:apikey-pro` | The write sets a first limit or raises one without `apikey-pro`. Nothing is stored. |
| `404` | `api key not found` | No key with that id in your tenant. |
| `429` | `monthly request limit (<n>) exceeded` | The key used its limit for this month. |

The 402 body in full:

```json
{"error": "payment_required", "plugin": "apikey", "feature": "feature:apikey-pro", "upgrade_url": ""}
```

The other errors of the key routes are on [API keys](api-keys.md#errors).

## Related

- [API keys](api-keys.md): scopes, roles, expiry and each key's request history.
- [Usage and quotas](usage-and-quotas.md): the counts behind the limit, and
  billing snapshots.
- [Tenant quotas](tenant-quotas.md): limits on a whole tenant.
- [Rate limiting](rate-limiting.md): limits per second and per minute.
