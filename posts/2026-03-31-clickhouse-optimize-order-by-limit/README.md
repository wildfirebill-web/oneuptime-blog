# How to Optimize ORDER BY with LIMIT in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ORDER BY, LIMIT, Query Optimization, Sorting, Performance

Description: Optimize ClickHouse ORDER BY LIMIT queries by leveraging primary key order, using LIMIT BY, and avoiding expensive full sorts.

---

`ORDER BY ... LIMIT` queries are common in ClickHouse for top-N analytics, but they can be expensive when ClickHouse must sort large intermediate result sets. Several techniques reduce the cost.

## Align ORDER BY with Primary Key

When the ORDER BY in your query matches the table's ORDER BY (primary key), ClickHouse can read data in sorted order without a full sort:

```sql
-- Table ordered by (region, event_time)
CREATE TABLE events (
  region LowCardinality(String),
  event_time DateTime,
  user_id UInt64,
  amount Decimal64(2)
) ENGINE = MergeTree()
ORDER BY (region, event_time);

-- FAST: aligned with primary key order - no sort needed
SELECT region, event_time, user_id
FROM events
WHERE region = 'US'
ORDER BY event_time DESC
LIMIT 100;
```

## Use read_in_order Optimization

ClickHouse automatically enables read-in-order optimization when ORDER BY matches the primary key. Verify it's active:

```sql
EXPLAIN PIPELINE
SELECT region, event_time
FROM events
ORDER BY event_time DESC
LIMIT 100;
-- Should show: MergingSortedTransform (not FullSortingMerge)
```

## Prefer LIMIT BY for Per-Group Top-N

`LIMIT BY` is more efficient than a subquery with window functions for per-group top-N queries:

```sql
-- Get top 3 products by revenue per region
SELECT region, product_id, revenue
FROM sales
ORDER BY region ASC, revenue DESC
LIMIT 3 BY region;
```

This is much faster than:

```sql
-- Expensive: full sort then filter
SELECT * FROM (
  SELECT region, product_id, revenue,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY revenue DESC) AS rn
  FROM sales
)
WHERE rn <= 3;
```

## Avoid Sorting Large Intermediate Results

Push filters before ORDER BY to reduce the number of rows sorted:

```sql
-- SLOW: filters after sort
SELECT * FROM events
ORDER BY event_time DESC
LIMIT 100
WHERE region = 'US';  -- Not valid SQL but shows the concept

-- FAST: filter reduces rows before sort
SELECT * FROM (
  SELECT * FROM events WHERE region = 'US'
)
ORDER BY event_time DESC
LIMIT 100;
```

Or equivalently:

```sql
SELECT * FROM events
WHERE region = 'US'
ORDER BY event_time DESC
LIMIT 100;
```

## Use max_rows_to_sort Setting

Protect against accidental large sorts:

```sql
SET max_rows_to_sort = 100000000;  -- Fail if sorting >100M rows
SET sort_overflow_mode = 'throw';  -- Throw error instead of breaking
```

## Optimize with Projections

Create a projection sorted by the query's ORDER BY columns:

```sql
ALTER TABLE events
  ADD PROJECTION top_users_proj (
    SELECT region, user_id, sum(amount) AS total
    GROUP BY region, user_id
    ORDER BY region, total DESC
  );

ALTER TABLE events MATERIALIZE PROJECTION top_users_proj;
```

## Summary

`ORDER BY LIMIT` performance in ClickHouse is best when the sort order aligns with the primary key (enabling read-in-order), using `LIMIT BY` for per-group top-N instead of window functions, filtering data before sorting, and using projections to maintain pre-sorted aggregated results. Monitor for expensive full sorts using `EXPLAIN PIPELINE`.
