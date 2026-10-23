---
title: Web application firewall
description: Block SQL injection, cross-site scripting, path traversal and other attacks before they reach the API, tune the rules, and review what was caught.
sidebar:
  order: 77
---

> **Requires** a license with the `waf` feature. See [pricing](https://lyeve.com/pricing/).

The web application firewall inspects every request to both APIs and blocks
the ones that look like attacks. It ships with 37 built-in rules, scores each
request by how many rules it trips and how serious they are, and logs every
match so you can see what it caught and tune what it gets wrong.

:::tip[Best for]
- **An instance on the public internet.** SQL injection, cross-site
  scripting, path traversal, command injection, server-side request forgery
  and tenant injection are refused before a handler runs.
- **Blocking a path you retired.** Your own regular expression, for your
  tenant, blocks or logs anything that matches.
- **Catching a probe that trips several small rules.** Severities add up, so
  several low signals can block a request no single rule would.
- **Seeing who is probing you.** Every match is logged with the rule, the
  path, the client address and the user agent.
:::

:::note[Use something else when]
- You want to limit request volume. Use [rate limiting](rate-limiting.md).
- You want to stop password guessing. See [captcha](captcha.md) and
  [trusted devices](trusted-devices.md).
:::

## How it works

The path, query string, headers, cookies and body are inspected. Encoded input
is decoded first, and JSON, form and multipart bodies are read field by field.
Each match adds to the request's score by the rule's severity:

| Severity | Score |
|---|---|
| `critical` | 5 |
| `high` | 3 |
| `medium` | 2 |
| `low` | 1 |

A request is blocked when a rule with the action `block` matches, or when the
total score is above the anomaly threshold (`5` by default). A match that does
not block is still logged.

These are not inspected:

- Sign-in routes under `/api/admin/auth/`, the firewall's own routes under
  `/api/admin/waf/`, and first-run setup at `/api/admin/setup`.
- The body of a write (`POST`, `PUT`, `PATCH`) to `/api/admin/content`,
  `/api/admin/content/*` or `/api/v1/content/*`. A content entry is stored as
  data, never run as a query. The path, query string, headers and cookies of
  those requests are still inspected.
- A body beyond `max_inspect_body_bytes`, 128 KiB by default. The rest of the
  body still reaches the API.

There is no switch that turns inspection off. To stop a rule from blocking,
set it to log only.

## Turn it on

1. Run with a license that carries `waf`, and restart the instance. The
   built-in rules apply from that start. See
   [licensing and tiers](../concepts/licensing-and-tiers.md).
2. Watch the violation log for a few days, and add exceptions for false
   positives before you tighten anything.

If you set `LYEVE_PLUGINS`, include `waf` in it.

## Try it

Admin console v0.21.0 shows whether the firewall is enforcing under **Settings > Security**. It
has no screen for the rules, the violation log, exceptions or settings, so the steps on this page
use the engine API.

You need an admin token in `TOKEN`. The
[quickstart](../getting-started/quickstart.md) shows how to get one.

1. List the rules:

   ```bash
   curl http://localhost:3001/api/admin/waf/rules \
     -H "Authorization: Bearer $TOKEN"
   ```

   Each rule has an `id` such as `SQLI-001`, a `name`, a `category`, a
   `severity`, an `action` (`block` or `log`), the `targets` it reads and
   whether it is `enabled`.

2. Send a request that looks like SQL injection:

   ```bash
   curl -i "http://localhost:3002/api/v1/content/post?q=1%20UNION%20SELECT%20password%20FROM%20users"
   ```

   ```http
   HTTP/1.1 403 Forbidden
   X-Waf-Block: true
   Content-Type: application/json

   {"code": "waf_blocked", "error": "request blocked by WAF", "matches": 1, "score": 5}
   ```

3. Find it in the violation log:

   ```bash
   curl "http://localhost:3001/api/admin/waf/violations?category=sqli&limit=1" \
     -H "Authorization: Bearer $TOKEN"
   ```

   ```json
   {
     "data": [
       {
         "id": 9,
         "timestamp": "2026-10-01T11:02:19Z",
         "rule_id": "SQLI-001",
         "rule_name": "UNION SELECT injection",
         "category": "sqli",
         "severity": "critical",
         "action": "block",
         "target": "query_string",
         "param": "query_string",
         "value": "q=1 UNION SELECT password FROM users",
         "path": "/api/v1/content/post",
         "method": "GET",
         "client_ip": "203.0.113.7",
         "tenant_id": "default",
         "user_agent": "curl/8.5.0"
       }
     ],
     "limit": 1,
     "offset": 0,
     "total_count": 1
   }
   ```

4. Read the summary:

   ```bash
   curl http://localhost:3001/api/admin/waf/stats \
     -H "Authorization: Bearer $TOKEN"
   ```

   The answer holds `total`, `by_category`, `by_severity`, `top_rules`,
   `last_24h`, `total_rules` and `enabled_rules`.

## Tune a rule

Set a rule to log only, or turn it off:

```bash
curl -X PUT http://localhost:3001/api/admin/waf/rules/SQLI-003 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"action": "log"}'
```

A change to a built-in rule is saved as an override for your tenant.

## Add your own rule

```bash
curl -X POST http://localhost:3001/api/admin/waf/rules \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "CUSTOM-001",
    "name": "Block the legacy export path",
    "description": "The old export endpoint was removed",
    "category": "custom",
    "severity": "high",
    "action": "block",
    "pattern": "(?i)/legacy-export",
    "targets": ["path"],
    "enabled": true
  }'
```

The answer is `201` with the rule. A custom rule applies to requests of the
tenant that created it, and a tenant can hold up to 500. A custom rule whose
`id` matches a built-in one replaces it for the tenant.

| Field | Values |
|---|---|
| `id` | Required. Use an id of your own, because a second rule with the same id is not refused. |
| `category` | `sqli`, `xss`, `path_traversal`, `tenant_injection`, `command_injection`, `ssrf` or `custom` |
| `severity` | `critical`, `high`, `medium` or `low`. Any other value scores 0. |
| `action` | `block` or `log` |
| `pattern` | A Go regular expression |
| `targets` | Any of `path`, `query_string`, `body`, `headers` and `cookie` |

A pattern that does not compile is saved but never matches, so test it before
you rely on it.

## Add an exception

When a rule matches traffic it should not, exempt that rule on a path:

```bash
curl -X POST http://localhost:3001/api/admin/waf/false-positives \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"rule_id": "XSS-002", "path_glob": "/api/v1/forms/*", "param_name": "*", "reason": "Support form accepts HTML"}'
```

In `path_glob`, `*` matches any run of characters, slashes included.
`param_name` is compared with the `param` of a match: `*` matches any, a
header or cookie name matches that header or cookie, and for the path, query
string and body the `param` is the target itself (`path`, `query_string` or
`body`). Delete an exception with `DELETE /api/admin/waf/false-positives/{id}`.

## Clean up the log

`DELETE /api/admin/waf/violations?older_than_hours=720` deletes old entries
and answers `{"pruned": <count>}`. Without a valid value it deletes entries
older than 30 days. The list takes `rule_id`, `category`, `severity`, `since`,
and `limit` (default 50, at most 500).

## Settings

Settings apply to the whole instance, so only a super admin changes them:

```bash
curl -X PUT http://localhost:3001/api/admin/waf/config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"anomaly_threshold": 8, "max_inspect_body_bytes": 131072}'
```

| Setting | What it does | Default |
|---|---|---|
| `anomaly_threshold` | A request whose score is above this is blocked. `0` becomes `5`, and a negative value is refused. | `5` |
| `max_inspect_body_bytes` | How much of a body is inspected. `0` takes the default, `-1` inspects whole bodies. | `131072` |
| `max_violation_body_length` | Stored and returned. A logged value is cut to 200 characters whatever it says. | `500` |

Inspecting whole bodies costs CPU on requests that need no credential. Raise
the limit with care.

Saved settings apply again after a restart. `GET /api/admin/waf/config`
shows the ones in force.

:::caution
If the license lapses, the firewall's routes answer `402`, but inspection and
blocking go on with the rules already loaded.
:::

## Routes

<details>
<summary>Firewall routes</summary>

| Method | Path | Role | Purpose |
|---|---|---|---|
| `GET` | `/api/admin/waf/rules` | `admin` | List rules |
| `POST` | `/api/admin/waf/rules` | `admin` | Add a custom rule. `201` |
| `GET` | `/api/admin/waf/rules/{id}` | `admin` | Read a rule |
| `PUT` | `/api/admin/waf/rules/{id}` | `admin` | Change a rule's `action` or `enabled` |
| `GET` | `/api/admin/waf/violations` | `admin` | List violations, paginated |
| `DELETE` | `/api/admin/waf/violations` | `admin` | Delete old violations |
| `GET` | `/api/admin/waf/false-positives` | `admin` | List exceptions |
| `POST` | `/api/admin/waf/false-positives` | `admin` | Add an exception |
| `DELETE` | `/api/admin/waf/false-positives/{id}` | `admin` | Remove an exception. `204` |
| `GET` | `/api/admin/waf/config` | `admin` | Read the settings |
| `PUT` | `/api/admin/waf/config` | `super_admin` | Change the settings |
| `GET` | `/api/admin/waf/stats` | `admin` | Violation statistics |

</details>

## Errors

| Status | Message | Cause |
|---|---|---|
| `403` | `request blocked by WAF` | The firewall blocked the request. |
| `404` | `The requested endpoint does not exist.` | The instance started without a license that carries `waf`. |
| `402` | `payment_required` | The license stopped carrying `waf` while the instance was running. |

<details>
<summary>Every other error</summary>

| Status | Message | Cause |
|---|---|---|
| `400` | `rule.id is required` | A custom rule had no id. |
| `409` | `maximum custom rules (500) reached` | The tenant has 500 custom rules. |
| `404` | `rule not found` | No rule with that id. |
| `400` | `rule_id is required` | An exception named no rule. |
| `403` | `only a super admin may scope a false positive outside its own tenant` | An admin set another tenant on an exception. |

</details>

## Related

- [Rate limiting](rate-limiting.md): limits on request volume.
- [Captcha](captcha.md): a challenge after failed sign-ins.
- [Harden your instance](../how-to/secure-your-instance.md): the production
  settings.
