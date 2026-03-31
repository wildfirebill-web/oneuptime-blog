# How to Delete Rows Based on Another Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Delete, Join, Subquery, DML

Description: Learn how to delete rows from a MySQL table using conditions or values from another table with JOIN-based DELETE and correlated subqueries.

---

## Why Delete Based on Another Table

Real-world cleanup tasks frequently require deleting rows from one table based on data in another. Examples include removing orders for banned customers, purging logs for deleted users, or cleaning up orphaned records after a migration.

## Method 1: Subquery with IN

The simplest approach uses a subquery with `IN` to identify rows to delete:

```sql
DELETE FROM orders
WHERE customer_id IN (
  SELECT id FROM customers WHERE status = 'banned'
);
```

MySQL rewrites this as a semi-join internally. It works well for moderate dataset sizes.

## Method 2: NOT IN for Orphan Cleanup

Delete rows that have no matching record in a parent table:

```sql
DELETE FROM order_items
WHERE order_id NOT IN (
  SELECT id FROM orders
);
```

Caution: `NOT IN` returns no rows if the subquery contains any `NULL` values. Use `NOT EXISTS` for safety:

```sql
DELETE FROM order_items oi
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.id = oi.order_id
);
```

## Method 3: JOIN-Based DELETE

The most performant option for large tables is a multi-table `DELETE` with an explicit `JOIN`:

```sql
DELETE oi
FROM order_items oi
JOIN orders o ON oi.order_id = o.id
JOIN customers c ON o.customer_id = c.id
WHERE c.status = 'banned';
```

Only list `oi` after `DELETE` to remove rows from `order_items` only. The joined tables are used purely for filtering.

## Deleting from Multiple Tables at Once

You can delete from several tables in a single statement by listing them after `DELETE`:

```sql
DELETE o, oi
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN customers c ON o.customer_id = c.id
WHERE c.status = 'banned';
```

This removes matching rows from both `orders` and `order_items` simultaneously.

## Using EXISTS for Correlated Filtering

```sql
DELETE FROM audit_logs al
WHERE EXISTS (
  SELECT 1
  FROM deleted_users du
  WHERE du.user_id = al.user_id
);
```

`EXISTS` stops as soon as one match is found per row, which can be faster than `IN` for large tables.

## Previewing Before Deleting

Always run a matching `SELECT` first:

```sql
SELECT oi.id, oi.order_id, c.status
FROM order_items oi
JOIN orders o ON oi.order_id = o.id
JOIN customers c ON o.customer_id = c.id
WHERE c.status = 'banned';
```

Verify the row count and spot-check the data before converting to a `DELETE`.

## Summary

MySQL offers several ways to delete rows based on another table: `IN` with a subquery for simplicity, `NOT EXISTS` for orphan cleanup, and JOIN-based multi-table `DELETE` for performance. For large-scale cleanups, use the JOIN approach and always preview with a matching `SELECT` to avoid unintended data loss.
