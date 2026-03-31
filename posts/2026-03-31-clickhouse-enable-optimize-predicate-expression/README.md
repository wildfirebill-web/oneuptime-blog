# How to Use enable_optimize_predicate_expression in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, enable_optimize_predicate_expression, Query Optimization, Predicate Pushdown, Performance

Description: Learn how enable_optimize_predicate_expression controls predicate pushdown in ClickHouse, allowing WHERE filters to be applied closer to the data source.

---

Predicate pushdown is one of the most impactful query optimizations in any analytical database. In ClickHouse, `enable_optimize_predicate_expression` controls whether the query optimizer pushes WHERE clause filters into subqueries and views, reducing the amount of data read and processed.

## What Predicate Pushdown Does

Without predicate pushdown, a query like this reads all rows from a subquery and then filters:

```sql
SELECT *
FROM (
    SELECT user_id, event_type, event_time
    FROM events
) AS sub
WHERE event_time >= today() - 7;
```

With `enable_optimize_predicate_expression = 1` (the default), ClickHouse pushes the `WHERE` condition into the subquery, so the storage layer filters data before it reaches the outer query:

```sql
-- Equivalent optimized form
SELECT *
FROM (
    SELECT user_id, event_type, event_time
    FROM events
    WHERE event_time >= today() - 7  -- pushed inside
) AS sub;
```

This allows ClickHouse to skip granules in the MergeTree index, dramatically reducing I/O.

## Checking the Default Setting

```sql
SELECT name, value, description
FROM system.settings
WHERE name = 'enable_optimize_predicate_expression';
```

The default is `1` (enabled). You would only disable it for debugging or when the optimizer produces incorrect results with certain view definitions.

## Using EXPLAIN to See Predicate Pushdown

```sql
EXPLAIN
SELECT *
FROM (
    SELECT
        session_id,
        user_id,
        page_url,
        created_at
    FROM page_views
) AS pv
WHERE pv.created_at >= toDate('2026-01-01')
  AND pv.user_id > 1000
SETTINGS enable_optimize_predicate_expression = 1;
```

Compare the output with the setting disabled:

```sql
EXPLAIN
SELECT *
FROM (
    SELECT
        session_id,
        user_id,
        page_url,
        created_at
    FROM page_views
) AS pv
WHERE pv.created_at >= toDate('2026-01-01')
  AND pv.user_id > 1000
SETTINGS enable_optimize_predicate_expression = 0;
```

The first output will show the filter applied at the `ReadFromMergeTree` stage; the second will show it applied later in the pipeline.

## Predicate Pushdown with Materialized Views

When querying through materialized views, predicate pushdown ensures filters propagate into the underlying table scan:

```sql
CREATE MATERIALIZED VIEW daily_summary
ENGINE = SummingMergeTree()
ORDER BY (date, category)
AS
SELECT
    toDate(event_time) AS date,
    category,
    count() AS event_count,
    sum(revenue) AS total_revenue
FROM events
GROUP BY date, category;

-- This query benefits from predicate pushdown
SELECT date, sum(total_revenue)
FROM daily_summary
WHERE date >= today() - 30
GROUP BY date
SETTINGS enable_optimize_predicate_expression = 1;
```

## Performance Impact

```sql
SELECT
    query_id,
    read_rows,
    read_bytes,
    query_duration_ms
FROM system.query_log
WHERE query LIKE '%page_views%'
  AND type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 10;
```

Compare `read_rows` and `read_bytes` between queries with and without the optimization. Effective predicate pushdown typically reduces both significantly.

## When to Disable It

Disable `enable_optimize_predicate_expression` temporarily when:

- Debugging unexpected query results that might be caused by over-aggressive optimization.
- Working around a known optimizer bug in a specific ClickHouse version.

```sql
-- Disable for a specific query
SELECT *
FROM (SELECT * FROM events) AS e
WHERE e.event_date = today()
SETTINGS enable_optimize_predicate_expression = 0;
```

## Summary

`enable_optimize_predicate_expression` is a critical optimization setting that enables predicate pushdown in ClickHouse, moving WHERE filters closer to the storage layer. It is enabled by default and should remain so in virtually all production deployments. Use `EXPLAIN` to verify that filters are being pushed down, and disable it only temporarily for debugging purposes.
