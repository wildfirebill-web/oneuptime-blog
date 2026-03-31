# How to Use system.row_policies in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, system.row_policies, Row-Level Security, Access Control, Policy

Description: Learn how to use the system.row_policies table in ClickHouse to inspect and manage row-level security policies that filter data per user or role.

---

Row-level security in ClickHouse lets you restrict which rows a user or role can see in a table. The `system.row_policies` table provides a complete view of all active row policies, their conditions, and which users or roles they apply to.

## What Are Row Policies?

A row policy attaches a filter expression to a table. When a user subject to that policy queries the table, ClickHouse automatically appends the filter to the WHERE clause. This is transparent to the user and enforced at the server level.

## Viewing All Row Policies

```sql
SELECT
    name,
    database,
    table,
    restrictiveness,
    roles,
    condition_as_string
FROM system.row_policies
ORDER BY database, table;
```

## Creating a Row Policy

Before querying `system.row_policies`, here is how you create a policy:

```sql
CREATE ROW POLICY tenant_filter ON events
    FOR SELECT
    USING tenant_id = currentUser()
    TO tenant_user_role;
```

This ensures users with `tenant_user_role` only see rows where `tenant_id` matches their username.

## Inspect Policy Conditions

To review the filter conditions applied to specific tables:

```sql
SELECT
    name,
    table,
    condition_as_string,
    roles,
    restrictiveness
FROM system.row_policies
WHERE database = 'production'
ORDER BY table;
```

## Check Which Roles Are Affected

Policies can be applied to specific roles or to ALL users. To find permissive vs. restrictive policies:

```sql
SELECT
    name,
    restrictiveness,
    roles,
    condition_as_string
FROM system.row_policies
WHERE table = 'orders';
```

A `RESTRICTIVE` policy uses AND logic with other policies; a `PERMISSIVE` policy uses OR logic.

## Modifying a Row Policy

To update the filter condition on an existing policy:

```sql
ALTER ROW POLICY tenant_filter ON events
    FOR SELECT
    USING tenant_id = 'acme_corp'
    TO tenant_user_role;
```

Then verify the change:

```sql
SELECT name, condition_as_string
FROM system.row_policies
WHERE name = 'tenant_filter';
```

## Dropping a Row Policy

```sql
DROP ROW POLICY tenant_filter ON events;
```

After dropping, confirm removal:

```sql
SELECT count()
FROM system.row_policies
WHERE name = 'tenant_filter';
```

## Columns Reference

| Column | Description |
|---|---|
| `name` | Policy name |
| `database` | Database where the policy applies |
| `table` | Table where the policy applies |
| `condition_as_string` | The filter expression as SQL |
| `restrictiveness` | PERMISSIVE or RESTRICTIVE |
| `roles` | Roles or users the policy applies to |

## Combining Row Policies with Column Permissions

Row policies work alongside column-level grants. A user might be restricted to certain rows AND certain columns simultaneously:

```sql
GRANT SELECT(order_id, status) ON orders TO analyst_role;
CREATE ROW POLICY analyst_filter ON orders
    FOR SELECT
    USING region = 'eu'
    TO analyst_role;
```

## Summary

The `system.row_policies` table in ClickHouse gives you a centralized view of all row-level security policies. Use it to audit data access controls, verify filter conditions, and manage multi-tenant or compliance-driven access patterns. Combined with role-based access control, row policies provide fine-grained data isolation at the query level.
