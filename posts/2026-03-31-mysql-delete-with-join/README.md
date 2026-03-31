# How to Use DELETE with JOIN in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Delete, Join, Multi-Table, DML

Description: Learn MySQL's JOIN-based DELETE syntax to remove rows from one or multiple tables simultaneously using conditions that span related tables.

---

## The Multi-Table DELETE Syntax

MySQL supports joining multiple tables in a `DELETE` statement. This lets you filter rows using data from related tables without a subquery, and optionally delete from multiple tables at once.

The syntax has two forms:

**Form 1 - listing tables before FROM:**

```sql
DELETE t1, t2
FROM t1 JOIN t2 ON t1.id = t2.t1_id
WHERE t1.status = 'inactive';
```

**Form 2 - listing tables after DELETE:**

```sql
DELETE t1
FROM t1
INNER JOIN t2 ON t1.id = t2.t1_id
WHERE t2.category = 'archived';
```

Only rows from tables listed after `DELETE` are removed. The joined tables serve as filters unless also listed.

## Deleting from One Table Using a JOIN Filter

A common use case is deleting rows from one table based on a condition in another:

```sql
DELETE e
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
WHERE d.name = 'Temporary';
```

This deletes employees in the "Temporary" department without affecting the `departments` table.

## LEFT JOIN to Delete Orphaned Rows

Use `LEFT JOIN` with a `NULL` check to find and delete orphaned records:

```sql
DELETE c
FROM comments c
LEFT JOIN posts p ON c.post_id = p.id
WHERE p.id IS NULL;
```

This removes all comments whose associated post no longer exists.

## Deleting from Both Joined Tables

To remove rows from both tables simultaneously, list them both after `DELETE`:

```sql
DELETE o, oi
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'cancelled'
  AND o.created_at < '2025-01-01';
```

Both the cancelled orders and their line items are removed in one atomic operation.

## Joining Three Tables

Chain multiple joins to filter with more precision:

```sql
DELETE oi
FROM order_items oi
INNER JOIN orders o ON oi.order_id = o.id
INNER JOIN customers c ON o.customer_id = c.id
WHERE c.region = 'EU'
  AND o.created_at < '2023-01-01';
```

Only `order_items` rows are deleted; `orders` and `customers` remain untouched.

## Previewing with a Matching SELECT

Before running the `DELETE`, verify the rows using a matching `SELECT`:

```sql
SELECT e.id, e.name, d.name AS dept
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
WHERE d.name = 'Temporary';
```

Replace `SELECT e.id, e.name, d.name AS dept` with `DELETE e` once the results look correct.

## Restrictions

- `LIMIT` and `ORDER BY` are not supported in multi-table `DELETE`.
- You cannot delete from a table and select from it in the same subquery within the same statement.
- Aliases must be used consistently throughout the statement.

## Summary

`DELETE` with `JOIN` in MySQL provides a concise way to delete rows using related-table conditions without nested subqueries. It can target one or multiple tables at once, supports all `JOIN` types, and is generally faster than equivalent `IN`-subquery patterns on large datasets.
