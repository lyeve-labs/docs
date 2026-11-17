---
title: "16. Users, roles and permissions"
description: "Create accounts for the people in your tenant, change their roles and passwords, and write a rule that lets a role read trips and nothing else."
sidebar:
  order: 16
covers:
  console:
    - "/admin/users"
    - "/admin/settings/permissions"
  api:
    - "GET /api/admin/users"
    - "POST /api/admin/users"
    - "PUT /api/admin/users/{id}/roles"
    - "PUT /api/admin/users/{id}/password"
    - "DELETE /api/admin/users/{id}"
    - "GET /api/admin/permissions/limits"
    - "POST /api/admin/permissions"
verified:
  engine: v0.62.0
  console: v0.28.0
---

So far you have worked alone as a super admin. In this lesson you add a colleague, change
what they may do, and give the `viewer` role the trips and nothing else. Both screens take the
`super_admin` role. You need the `trip` content type from lesson 3 and a super admin's token
in `TOKEN`, as in [lesson 15](connected-editors.md).

[Users, roles and permissions](../guides/user/roles-and-permissions.md) says what each
built-in role may do.

## Create an account

> **Included free** on every install.

<div class="paths">
<div data-path="console">

1. Open **Access > Users** and choose **New user**.
2. Enter `sam@example.com` in **Email** and a password in **Password**. The password policy
   asks for at least 12 characters, with upper case, lower case and a digit.
3. On an engine with several tenants, pick the **Tenant** the account signs in to.
4. Under **Roles**, tick **editor** and choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "sam@example.com", "password": "Editor-Account-2026", "roles": ["editor"]}'
```

The answer is `201` with the new user. Copy its `id` into `USER_ID`.

</div>
</div>

![Opening New user, entering the email and password, ticking editor and creating the account.](./media/users-and-roles-new-user.webm)

The list has these controls:

| Control | What it does |
|---|---|
| Search and **Role** | Narrow the accounts on this page by email or role |
| **Edit roles**, the pencil | Change the account's roles |
| **Set password**, the key | Set a new password. Saving signs the person out of every session |
| **Delete**, the bin | Delete the account, after a question |

## Change a role and a password

<div class="paths">
<div data-path="console">

1. On the row for `sam@example.com`, choose the pencil, tick **admin** under **Roles** and
   choose **Save**.
2. Choose the key, enter a **New password** and keep **Remove this account's passkeys**
   ticked. Choose **Set password**.

</div>
<div data-path="api">

```bash
curl -X PUT http://localhost:3001/api/admin/users/$USER_ID/roles \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"roles": ["admin"]}'

curl -X PUT http://localhost:3001/api/admin/users/$USER_ID/password \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"password": "Admin-Account-2026"}'
```

A password that fails the policy answers `422`. Every session the person held ends.

</div>
</div>

## Let a role read trips and nothing else

> **Included free** on every install, with rules for up to three roles. More roles need a
> license with the `rbac-pro` feature.

A rule says what a role may do with one content type or flow. A role with no rule for a
content type is denied it.

<div class="paths">
<div data-path="console">

1. Open **Access > Permissions**.
2. Choose **New role**, enter `viewer` in **Role name** and choose **Create**. The role's row
   appears with an empty rule for every content type.
3. Choose the plus button on the `viewer` row. In **Resource**, pick `trip`.
4. Tick **Read**. To hide a field from this role, enter its name in **Field mask**.
5. Choose **Create**.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/permissions/limits \
  -H "Authorization: Bearer $TOKEN"

curl -X POST http://localhost:3001/api/admin/permissions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role": "viewer", "schema_name": "trip", "actions": ["read"]}'
```

Sending a rule for the same role and resource again replaces it.

</div>
</div>

![Creating the viewer role, adding a rule on trip and ticking Read.](./media/users-and-roles-rule.webm)

A rule with no action and no field mask changes nothing, and the form warns you, because it
still counts toward the role ceiling. **Show resources with no rule** lists the content types
and flows a role reaches only through its wildcard rule.

On a free install, once three roles hold rules, the page says **3 of 3 roles** and offers no
**New role**. Your existing roles keep working. With `rbac-pro` there is no ceiling.

![Permissions on a free install, with the role count at its ceiling.](./media/users-and-roles-permissions-free.png)

![Permissions on a licensed install, with no role ceiling.](./media/users-and-roles-permissions-licensed.png)

## Check the rule as that person

Create an account that holds only `viewer`, as in the first section, and sign in as it to get
its token in `VIEWER_TOKEN`. Then read trips on the Content API:

```bash
curl http://localhost:3002/api/v1/content/trip \
  -H "Authorization: Bearer $VIEWER_TOKEN"
```

The trips come back. Any content type without a rule for `viewer` answers `403`.

## Next

[17. Sign-in, SSO and provisioning](sign-in-and-sso.md) lets people sign in with the account
they already have. For depth, read [Access rules](../features/access-rules.md) and
[Users, roles and permissions](../guides/user/roles-and-permissions.md).
