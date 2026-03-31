# How to Use SHOW WARNINGS After EXPLAIN in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN, Query, Optimization, Performance

Description: Learn how to use SHOW WARNINGS after EXPLAIN in MySQL to see the rewritten query the optimizer actually executes, revealing hidden transformations.

---

## Why Use SHOW WARNINGS After EXPLAIN?

Running `EXPLAIN` on a query shows the execution plan, but MySQL's query optimizer often rewrites your SQL before executing it. These rewrites - such as subquery flattening, constant folding, or predicate pushdown - can significantly change how the query runs.

By running `SHOW WARNINGS` immediately after `EXPLAIN`, you can see the optimizer's rewritten version of your query. This reveals exactly what MySQL is executing, not just what you wrote.

## Basic Usage

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id IN (SELECT id FROM customers WHERE country = 'US');
SHOW WARNINGS;
```

The `SHOW WARNINGS` output will include a `Note` row with the rewritten query in the `Message` column.

## Example: Subquery Flattening

Consider this query with a subquery:

```sql
EXPLAIN
SELECT p.name, p.price
FROM products p
WHERE p.category_id IN (SELECT id FROM categories WHERE active = 1);
SHOW WARNINGS;
```

The warning message might show:

```sql
/* select#1 */
SELECT `p`.`name`, `p`.`price`
FROM `mydb`.`products` `p`
SEMI JOIN (`mydb`.`categories`)
WHERE (`p`.`category_id` = `mydb`.`categories`.`id`)
AND (`mydb`.`categories`.`active` = 1)
```

The optimizer converted your `IN (subquery)` into a `SEMI JOIN`, which is typically more efficient.

## Example: Constant Folding

```sql
EXPLAIN SELECT * FROM products WHERE price > 10 + 5;
SHOW WARNINGS;
```

Rewritten query:

```sql
/* select#1 */ SELECT `mydb`.`products`.`id`, ...
FROM `mydb`.`products`
WHERE (`mydb`.`products`.`price` > 15)
```

MySQL pre-computed `10 + 5` to `15` before execution.

## Practical Workflow

```sql
-- Step 1: Run EXPLAIN
EXPLAIN
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id;

-- Step 2: Immediately run SHOW WARNINGS
SHOW WARNINGS;
```

Check the output for:
- `Note 1003` - Contains the optimizer-rewritten query
- Other warning codes indicating issues like missing indexes or type conversions

## Warning Codes to Know

```text
Code 1003  - Optimizer's rewritten query (most common after EXPLAIN)
Code 1292  - Truncated incorrect datetime value
Code 1366  - Incorrect integer value
```

## Using EXPLAIN EXTENDED (MySQL 5.x)

In older MySQL 5.x versions, you must use `EXPLAIN EXTENDED` to trigger the note that `SHOW WARNINGS` reads:

```sql
-- MySQL 5.x only
EXPLAIN EXTENDED SELECT * FROM orders WHERE status = 'pending';
SHOW WARNINGS;
```

In MySQL 8.0, `EXPLAIN` alone is sufficient and `EXPLAIN EXTENDED` is deprecated.

## Identifying Implicit Type Conversions

One of the most valuable uses of `SHOW WARNINGS` is catching implicit type conversions that prevent index use:

```sql
EXPLAIN SELECT * FROM users WHERE phone_number = 5551234567;
SHOW WARNINGS;
-- Warning might reveal: phone_number is VARCHAR, 5551234567 is INT
-- MySQL must convert every row's phone_number to INT to compare
-- Result: full table scan despite having an index on phone_number
```

The fix is always to quote string values:

```sql
SELECT * FROM users WHERE phone_number = '5551234567';
```

## Summary

`SHOW WARNINGS` after `EXPLAIN` is an underused but powerful tool for understanding MySQL's optimizer behavior. It exposes query rewrites like subquery-to-semijoin conversions, constant folding, and predicate transformations. Use it whenever EXPLAIN shows an unexpected execution plan - the rewritten query often explains why the optimizer made a particular choice, and it helps you write SQL that more closely matches what MySQL actually executes.
