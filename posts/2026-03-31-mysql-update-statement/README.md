# How to Update Rows with UPDATE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Update, DML, Query, WHERE

Description: Learn the MySQL UPDATE statement syntax, how to use WHERE to target specific rows, and how to avoid common mistakes that overwrite your entire table.

---

## The UPDATE Statement

`UPDATE` modifies existing rows in a table. The basic syntax is:

```sql
UPDATE table_name
SET column1 = value1, column2 = value2
WHERE condition;
```

Without a `WHERE` clause, every row in the table is updated. Always include a `WHERE` clause unless a full-table update is intentional.

## Updating a Single Row by Primary Key

The most common and safest pattern targets a row by its primary key:

```sql
UPDATE users
SET email = 'alice@newdomain.com'
WHERE id = 42;
```

Because `id` is unique, at most one row changes. Verify the result with `ROW_COUNT()`:

```sql
SELECT ROW_COUNT();
```

## Updating Multiple Rows with a Condition

To update all rows matching a condition, provide a broader `WHERE` clause:

```sql
UPDATE products
SET in_stock = 0
WHERE quantity = 0;
```

This sets `in_stock` to `0` for every product with zero quantity. MySQL executes this as a single statement regardless of how many rows match.

## Using Expressions in SET

The `SET` clause can use arithmetic expressions, string functions, or even reference the current column value:

```sql
UPDATE accounts
SET balance = balance - 100.00
WHERE id = 7;
```

This deducts 100 from the current balance rather than replacing it with a fixed value.

## Safe Update Mode

MySQL Workbench and some connectors enable `sql_safe_updates` by default, which blocks `UPDATE` without a `WHERE` clause or a primary key lookup:

```sql
SET sql_safe_updates = 0;

UPDATE products SET discount = 0.05;  -- updates all rows

SET sql_safe_updates = 1;
```

Disable it only when you truly intend a full-table update.

## Previewing Changes Before Updating

Use a matching `SELECT` statement to preview which rows will be affected before committing the change:

```sql
SELECT id, status
FROM orders
WHERE created_at < '2025-01-01' AND status = 'pending';
```

Once confident, convert it to an `UPDATE`:

```sql
UPDATE orders
SET status = 'archived'
WHERE created_at < '2025-01-01' AND status = 'pending';
```

## Updating with a Subquery

Reference a subquery to compute dynamic values:

```sql
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = (
  SELECT id FROM departments WHERE name = 'Engineering'
);
```

Note that MySQL does not allow you to update a table and select from it in the same subquery in a single statement. Use a derived table as a workaround if needed.

## Checking Affected Rows

`ROW_COUNT()` returns the number of rows actually changed (not just matched). If the new value equals the existing value, the row is matched but not changed:

```sql
UPDATE users SET active = 1 WHERE active = 1;
SELECT ROW_COUNT(); -- returns 0
```

## Summary

The MySQL `UPDATE` statement is straightforward but requires care. Always use a `WHERE` clause to limit the scope, use primary key lookups when possible, preview changes with a matching `SELECT` first, and check `ROW_COUNT()` to confirm that the expected number of rows was modified.
