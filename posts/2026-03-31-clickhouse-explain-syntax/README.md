# How to Use EXPLAIN SYNTAX in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, EXPLAIN, Syntax, Query Rewrite

Description: Learn how EXPLAIN SYNTAX in ClickHouse shows the canonical form of a query after alias expansion, wildcard expansion, and normalization.

---

`EXPLAIN SYNTAX` in ClickHouse outputs the canonical form of your SQL query after the parser and normalizer have processed it, but before the query planner runs. It expands wildcards, resolves aliases, folds constants, and rewrites shorthand syntax into explicit form. This makes it invaluable for understanding what query ClickHouse will actually execute when you write shorthand SQL.

## Basic EXPLAIN SYNTAX Syntax

```sql
EXPLAIN SYNTAX
SELECT *
FROM events
WHERE event_date = today() - 7;
```

Sample output:

```text
SELECT
    event_id,
    user_id,
    event_name,
    event_date,
    value
FROM events
WHERE event_date = (today() - 7)
```

The wildcard `*` is expanded into the actual column list, and the date arithmetic is preserved as an expression.

## Wildcard Expansion

One of the most common uses of EXPLAIN SYNTAX is to see what `SELECT *` expands to:

```sql
-- Table: orders(order_id, user_id, product_id, amount, status, created_at)
EXPLAIN SYNTAX
SELECT * EXCEPT (created_at)
FROM orders;
```

```text
SELECT
    order_id,
    user_id,
    product_id,
    amount,
    status
FROM orders
```

The `EXCEPT` clause is resolved and the remaining columns are listed explicitly. This helps you verify that exclusions work as expected before running the query.

## Alias Expansion and Resolution

ClickHouse resolves aliases used in WHERE or GROUP BY back to their expressions in the canonical form:

```sql
EXPLAIN SYNTAX
SELECT
    toStartOfHour(event_time) AS hour,
    count() AS cnt
FROM events
GROUP BY hour
HAVING cnt > 100;
```

```text
SELECT
    toStartOfHour(event_time) AS hour,
    count() AS cnt
FROM events
GROUP BY toStartOfHour(event_time)
HAVING count() > 100
```

Notice that `GROUP BY hour` is rewritten to `GROUP BY toStartOfHour(event_time)` and `HAVING cnt > 100` becomes `HAVING count() > 100`. This is the form ClickHouse actually executes.

## Constant Folding

Constant expressions are evaluated at normalization time:

```sql
EXPLAIN SYNTAX
SELECT
    user_id,
    amount * (1 + 0.08) AS amount_with_tax
FROM orders;
```

```text
SELECT
    user_id,
    amount * 1.08 AS amount_with_tax
FROM orders
```

The constant arithmetic `1 + 0.08` is folded to `1.08`. Knowing this can help you write cleaner SQL without worrying that repeated constant arithmetic harms performance.

## Subquery Normalization

Subqueries are also rewritten into their canonical form:

```sql
EXPLAIN SYNTAX
SELECT user_id
FROM users
WHERE user_id IN (
    SELECT user_id FROM orders WHERE total > 500
);
```

```text
SELECT user_id
FROM users
WHERE user_id IN (
    SELECT user_id
    FROM orders
    WHERE total > 500
)
```

For more complex queries with correlated subqueries or rewrite rules, EXPLAIN SYNTAX can reveal transformations that are not obvious from the original SQL.

## WITH Clause Expansion

Common table expressions (CTEs) written with `WITH` are inlined when referenced:

```sql
EXPLAIN SYNTAX
WITH active_users AS (
    SELECT user_id FROM users WHERE is_active = 1
)
SELECT o.order_id, o.total
FROM orders o
WHERE o.user_id IN (SELECT user_id FROM active_users);
```

```text
SELECT
    o.order_id,
    o.total
FROM orders AS o
WHERE o.user_id IN (
    SELECT user_id
    FROM users
    WHERE is_active = 1
)
```

The CTE is inlined into the subquery. Understanding this can help you reason about whether a CTE is evaluated once or multiple times.

## Practical Use: Verifying Query Rewrites

EXPLAIN SYNTAX is particularly useful before deploying a query rewrite to production:

```sql
-- Original query using shorthand date function
EXPLAIN SYNTAX
SELECT
    toDate(created_at) AS day,
    count() AS orders,
    sum(total) AS revenue
FROM orders
WHERE toYear(created_at) = 2024
GROUP BY day
ORDER BY day;
```

```text
SELECT
    toDate(created_at) AS day,
    count() AS orders,
    sum(total) AS revenue
FROM orders
WHERE toYear(created_at) = 2024
GROUP BY toDate(created_at)
ORDER BY toDate(created_at) ASC
```

You can see that `ORDER BY day` became `ORDER BY toDate(created_at) ASC` - confirming the alias was resolved and that ascending order is the default.

## Summary

`EXPLAIN SYNTAX` is a lightweight way to preview how ClickHouse normalizes and rewrites your SQL. It expands wildcards and aliases, folds constants, and inlines CTEs, giving you the exact form that is passed to the query planner. Use it whenever you write shorthand SQL to confirm that ClickHouse interprets your intent correctly, and to catch unexpected alias resolution or wildcard expansion before the query runs against production data.
