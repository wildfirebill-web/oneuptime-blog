# How to Optimize Correlated Subqueries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Correlated Subquery, Query Optimization, Performance, JOIN

Description: Eliminate correlated subqueries in ClickHouse by rewriting them as JOINs or window functions for dramatically better performance.

---

Correlated subqueries - those that reference the outer query - are one of the most common performance pitfalls in SQL. ClickHouse executes them row by row, making them extremely slow on large datasets.

## Identify Correlated Subqueries

A correlated subquery references a column from the outer query:

```sql
-- SLOW: correlated subquery - executes inner query for each outer row
SELECT
  user_id,
  amount,
  (SELECT avg(amount) FROM orders WHERE user_id = o.user_id) AS user_avg
FROM orders o
WHERE amount > (SELECT avg(amount) FROM orders WHERE user_id = o.user_id);
```

## Rewrite with a JOIN

Pre-aggregate the correlated result and JOIN it:

```sql
-- FAST: compute averages once, then join
WITH user_averages AS (
  SELECT
    user_id,
    avg(amount) AS avg_amount
  FROM orders
  GROUP BY user_id
)
SELECT
  o.user_id,
  o.amount,
  ua.avg_amount
FROM orders o
JOIN user_averages ua ON o.user_id = ua.user_id
WHERE o.amount > ua.avg_amount;
```

This computes averages once instead of once per row.

## Use Window Functions

Window functions are often the most elegant rewrite for correlated subqueries:

```sql
-- Using window function instead of correlated subquery
SELECT
  user_id,
  amount,
  avg(amount) OVER (PARTITION BY user_id) AS user_avg
FROM orders
WHERE amount > avg(amount) OVER (PARTITION BY user_id);
```

Note: Filtering on window functions requires a subquery or CTE:

```sql
SELECT *
FROM (
  SELECT
    user_id,
    amount,
    avg(amount) OVER (PARTITION BY user_id) AS user_avg
  FROM orders
)
WHERE amount > user_avg;
```

## Handle EXISTS Subqueries

Correlated EXISTS clauses are also slow:

```sql
-- SLOW
SELECT user_id FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.user_id AND o.amount > 1000
);

-- FAST: use a semi-join equivalent
SELECT DISTINCT u.user_id
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id AND o.amount > 1000;
```

## Verify the Rewrite with EXPLAIN

```sql
EXPLAIN PLAN
SELECT user_id FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.user_id
);
```

If the plan shows repeated subquery execution, the rewrite is necessary.

## Benchmark the Improvement

```sql
-- Before
SELECT count() FROM (
  SELECT user_id FROM users u
  WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = u.user_id)
);

-- After
SELECT count() FROM (
  SELECT DISTINCT u.user_id
  FROM users u
  JOIN orders o ON u.user_id = o.user_id
);
```

## Summary

Correlated subqueries in ClickHouse execute row-by-row and should be eliminated by rewriting as JOINs with pre-aggregated CTEs or as window functions. EXISTS correlated subqueries can become INNER JOINs with DISTINCT. These rewrites typically reduce execution time from minutes to seconds on large tables.
