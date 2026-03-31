# How to Optimize Subquery Performance in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Subquery, Query Optimization, Performance, JOIN

Description: Optimize subquery performance in ClickHouse by rewriting subqueries as JOINs, CTEs, or IN clauses with proper data structures.

---

Subqueries in ClickHouse can be expensive if not written carefully. ClickHouse's query optimizer has improved significantly, but understanding how subqueries execute helps you write faster queries.

## Understand How Subqueries Execute

ClickHouse executes most subqueries by materializing the inner result first, then using it in the outer query. For large inner results, this can be memory-intensive:

```sql
-- This materializes all user_ids matching the condition in memory
SELECT count()
FROM events
WHERE user_id IN (
  SELECT user_id FROM users WHERE country = 'US'
);
```

## Rewrite as a JOIN for Large Datasets

For large subqueries, an explicit JOIN with proper table placement is often faster:

```sql
-- Place the smaller table on the right side of JOIN
SELECT count()
FROM events e
INNER JOIN (
  SELECT user_id FROM users WHERE country = 'US'
) u ON e.user_id = u.user_id;
```

In ClickHouse, the right-side table of a JOIN is loaded entirely into memory as a hash table. Always put the smaller dataset on the right.

## Use CTEs for Readability and Reuse

Common Table Expressions (CTEs) allow reusing subquery results without re-executing:

```sql
WITH us_users AS (
  SELECT user_id FROM users WHERE country = 'US'
),
recent_events AS (
  SELECT user_id, event_type, event_time
  FROM events
  WHERE event_time >= now() - INTERVAL 7 DAY
)
SELECT u.user_id, count() AS event_count
FROM us_users u
JOIN recent_events e ON u.user_id = e.user_id
GROUP BY u.user_id
ORDER BY event_count DESC
LIMIT 100;
```

## Use IN with a Subquery on a Distributed Table

For distributed clusters, push down the subquery to avoid transferring large intermediate results:

```sql
SET distributed_product_mode = 'local';

SELECT count()
FROM distributed_events
WHERE user_id IN (
  SELECT user_id FROM distributed_users WHERE country = 'US'
);
```

## Avoid Subqueries in SELECT Clause

Scalar subqueries in the SELECT clause execute once per row, which is very slow:

```sql
-- SLOW: executes subquery for each row
SELECT
  user_id,
  (SELECT name FROM users WHERE id = e.user_id) AS user_name
FROM events e;

-- FAST: use a JOIN instead
SELECT e.user_id, u.name
FROM events e
LEFT JOIN users u ON e.user_id = u.id;
```

## Check Execution with EXPLAIN

Verify how your subquery will execute:

```sql
EXPLAIN PLAN
SELECT count()
FROM events
WHERE user_id IN (SELECT user_id FROM users WHERE country = 'US');
```

## Summary

Subquery performance in ClickHouse improves significantly by rewriting scalar subqueries as JOINs, keeping the smaller table on the right side of JOIN, using CTEs to avoid re-execution, and setting `distributed_product_mode = 'local'` on distributed clusters. Always verify execution plans with `EXPLAIN PLAN` after rewriting.
