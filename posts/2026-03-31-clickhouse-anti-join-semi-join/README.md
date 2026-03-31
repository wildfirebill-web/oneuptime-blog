# How to Use ANTI JOIN and SEMI JOIN in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, ANTI JOIN, SEMI JOIN, Filter

Description: Learn how to use LEFT/RIGHT ANTI JOIN and LEFT/RIGHT SEMI JOIN in ClickHouse to filter rows based on existence in another table.

---

`SEMI JOIN` and `ANTI JOIN` are filtering join types that return rows based on whether a match exists in another table, without actually including columns from the other table in the result. `SEMI JOIN` keeps rows that have a match; `ANTI JOIN` keeps rows that do not. ClickHouse supports `LEFT` and `RIGHT` variants of both, offering a cleaner and often faster alternative to `IN`, `NOT IN`, `EXISTS`, and `NOT EXISTS` subqueries.

## LEFT SEMI JOIN

`LEFT SEMI JOIN` returns rows from the left table that have at least one match in the right table. Only columns from the left table appear in the result, and duplicate matches on the right side do not produce duplicate rows on the left.

```sql
-- Users who have placed at least one order
SELECT u.user_id, u.name, u.email
FROM users AS u
LEFT SEMI JOIN orders AS o ON u.user_id = o.user_id;
```

This is equivalent to the `IN` subquery form but typically more efficient:

```sql
-- Equivalent using IN (less efficient for large right-side tables)
SELECT user_id, name, email
FROM users
WHERE user_id IN (SELECT user_id FROM orders);
```

## RIGHT SEMI JOIN

`RIGHT SEMI JOIN` returns rows from the right table that have at least one match in the left table. Only columns from the right table are returned.

```sql
-- Orders that belong to a known user (filter out orphaned orders)
SELECT o.order_id, o.amount, o.status
FROM users AS u
RIGHT SEMI JOIN orders AS o ON u.user_id = o.user_id;
```

## LEFT ANTI JOIN

`LEFT ANTI JOIN` returns rows from the left table that have no match in the right table. This is the "set difference" operation - the SQL equivalent of `NOT IN` or `NOT EXISTS`.

```sql
-- Users who have never placed an order
SELECT u.user_id, u.name, u.email
FROM users AS u
LEFT ANTI JOIN orders AS o ON u.user_id = o.user_id;
```

Compare with the `NOT IN` equivalent:

```sql
-- Equivalent using NOT IN (can have unexpected behavior with NULLs)
SELECT user_id, name, email
FROM users
WHERE user_id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL);
```

`LEFT ANTI JOIN` handles NULLs correctly and avoids the NULL-pitfall of `NOT IN`.

## RIGHT ANTI JOIN

`RIGHT ANTI JOIN` returns rows from the right table that have no match in the left table.

```sql
-- Orders that reference a user_id not present in the users table (orphaned records)
SELECT o.order_id, o.user_id, o.amount
FROM users AS u
RIGHT ANTI JOIN orders AS o ON u.user_id = o.user_id;
```

## Combining ANTI JOIN with Additional Filters

The `ON` condition can be extended with `WHERE` clauses to add further filtering after the anti-join:

```sql
-- Premium users who have no completed orders in the last 90 days
SELECT u.user_id, u.name, u.tier
FROM users AS u
LEFT ANTI JOIN (
    SELECT DISTINCT user_id
    FROM orders
    WHERE status = 'completed'
      AND order_date >= today() - 90
) AS recent_orders ON u.user_id = recent_orders.user_id
WHERE u.tier = 'premium';
```

## Performance Comparison

`SEMI JOIN` and `ANTI JOIN` are usually faster than their `IN`/`NOT IN` equivalents for large right-side tables because ClickHouse only needs to build a membership structure (not a full join result), and stops checking the right table once the first match is found for `SEMI JOIN`.

```sql
-- Benchmark: ANTI JOIN vs NOT IN for a large exclusion list

-- Option 1: LEFT ANTI JOIN (recommended)
SELECT count()
FROM events AS e
LEFT ANTI JOIN blocked_users AS b ON e.user_id = b.user_id;

-- Option 2: NOT IN subquery (avoid for large tables)
SELECT count()
FROM events
WHERE user_id NOT IN (SELECT user_id FROM blocked_users);
```

## Practical Example - Churn Detection

Identify users who were active in the previous month but not in the current month:

```sql
WITH
    prev_month_active AS (
        SELECT DISTINCT user_id
        FROM events
        WHERE toStartOfMonth(event_time) = toStartOfMonth(today() - INTERVAL 1 MONTH)
    ),
    curr_month_active AS (
        SELECT DISTINCT user_id
        FROM events
        WHERE toStartOfMonth(event_time) = toStartOfMonth(today())
    )
SELECT p.user_id
FROM prev_month_active AS p
LEFT ANTI JOIN curr_month_active AS c ON p.user_id = c.user_id;
```

## Summary

ClickHouse provides `LEFT SEMI JOIN`, `RIGHT SEMI JOIN`, `LEFT ANTI JOIN`, and `RIGHT ANTI JOIN` as first-class join types for existence-based filtering. `SEMI JOIN` retains rows with a match; `ANTI JOIN` retains rows without a match. Both avoid producing extra rows from right-side duplicates and are generally more efficient and NULL-safe than equivalent `IN`, `NOT IN`, `EXISTS`, or `NOT EXISTS` subqueries.
