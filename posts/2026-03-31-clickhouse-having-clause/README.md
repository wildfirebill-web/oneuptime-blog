# How to Use HAVING Clause in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, HAVING, Aggregation, Filter

Description: Understand how to use HAVING to filter aggregate results in ClickHouse, and when to use it versus WHERE for optimal query performance.

---

The `HAVING` clause filters rows after aggregation, making it the right tool when you need to exclude groups based on an aggregate value such as a sum, count, or average. In ClickHouse - a system designed for high-throughput analytical queries - understanding the distinction between `WHERE` and `HAVING` is important both for correctness and for performance. This post explains the HAVING clause with practical examples.

## WHERE vs HAVING

The key difference: `WHERE` filters individual rows before grouping, while `HAVING` filters groups after aggregation.

```sql
-- WHERE filters rows before aggregation (efficient - reduces input rows)
SELECT
    user_id,
    sum(value) AS total_spent
FROM events
WHERE event_type = 'buy'         -- applied before GROUP BY
GROUP BY user_id
HAVING total_spent > 500;        -- applied after GROUP BY
```

Using `WHERE` for pre-group conditions is always faster because it reduces the number of rows the aggregation engine must process.

### What Goes in WHERE vs HAVING

```sql
-- Correct: filter on a raw column in WHERE, aggregate result in HAVING
SELECT
    event_type,
    count()    AS cnt,
    avg(value) AS avg_value
FROM events
WHERE created_at >= today() - 30   -- row-level filter in WHERE
GROUP BY event_type
HAVING avg_value > 10.0;           -- aggregate filter in HAVING
```

```sql
-- Incorrect: you cannot reference an aggregate in WHERE
-- This will cause an error:
-- SELECT event_type, count() AS cnt FROM events
-- WHERE cnt > 100           -- ERROR: aggregate not allowed in WHERE
-- GROUP BY event_type;
```

## Filtering on Aggregate Functions

HAVING accepts any aggregate expression, not just aliases defined in SELECT.

```sql
-- Filter using sum() directly in HAVING
SELECT
    user_id,
    count()    AS orders,
    sum(value) AS revenue
FROM events
WHERE event_type = 'buy'
GROUP BY user_id
HAVING sum(value) > 1000
ORDER BY revenue DESC;
```

```sql
-- Find categories with more than 50 events and an average value under 5
SELECT
    event_type,
    count()    AS cnt,
    avg(value) AS avg_value
FROM events
GROUP BY event_type
HAVING cnt > 50
   AND avg_value < 5.0;
```

## HAVING with Multiple Aggregate Conditions

You can combine multiple aggregate conditions in HAVING using AND and OR.

```sql
SELECT
    toDate(created_at) AS event_date,
    event_type,
    count()            AS cnt,
    sum(value)         AS total_value,
    max(value)         AS max_value
FROM events
GROUP BY event_date, event_type
HAVING cnt >= 10
   AND total_value > 100.0
   AND max_value < 10000.0
ORDER BY event_date DESC, total_value DESC;
```

## HAVING with GROUP BY Modifiers

HAVING works with GROUP BY modifiers like WITH ROLLUP and WITH TOTALS. Note that HAVING is applied after the modifier generates subtotal rows, so subtotal rows can also be filtered out.

```sql
SELECT
    event_type,
    toDate(created_at) AS event_date,
    sum(value)         AS revenue
FROM events
GROUP BY event_type, event_date
WITH ROLLUP
HAVING revenue > 500
ORDER BY event_type, event_date;
```

## Practical Example: Top Active Users

```sql
-- Find users with at least 5 purchases totalling more than $200
-- in the past 90 days
SELECT
    user_id,
    count()        AS purchase_count,
    sum(value)     AS total_spent,
    avg(value)     AS avg_order_value,
    max(value)     AS largest_order
FROM events
WHERE event_type = 'buy'
  AND created_at >= now() - INTERVAL 90 DAY
GROUP BY user_id
HAVING purchase_count >= 5
   AND total_spent > 200.0
ORDER BY total_spent DESC
LIMIT 50;
```

## Performance Tip: Push Conditions to WHERE When Possible

If a HAVING condition does not depend on an aggregate, move it to WHERE.

```sql
-- Less efficient: event_type filter in HAVING
SELECT event_type, count() AS cnt
FROM events
GROUP BY event_type
HAVING event_type = 'buy' AND cnt > 100;

-- More efficient: event_type filter in WHERE
SELECT event_type, count() AS cnt
FROM events
WHERE event_type = 'buy'
GROUP BY event_type
HAVING cnt > 100;
```

## Summary

`HAVING` is the correct clause for filtering on aggregate results in ClickHouse, while `WHERE` handles row-level filtering before grouping. Always prefer `WHERE` for non-aggregate conditions to reduce the data processed by the aggregation engine. Combining both clauses - `WHERE` for selective pre-group filtering and `HAVING` for post-group conditions - produces the most efficient analytical queries.
