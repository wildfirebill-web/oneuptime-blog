# How to Implement Pagination in ClickHouse Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Pagination, LIMIT OFFSET, Keyset Pagination, Cursor

Description: Learn how to implement efficient pagination in ClickHouse using LIMIT/OFFSET, keyset-based cursor pagination, and row_number for large result sets.

---

## Pagination in ClickHouse

ClickHouse is optimized for analytical batch queries, but pagination is still needed for APIs and dashboards. Standard LIMIT/OFFSET works for small datasets, while keyset pagination scales to billions of rows without performance degradation.

## Basic LIMIT/OFFSET Pagination

```sql
-- Page 1 (rows 1-20)
SELECT user_id, name, created_at
FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- Page 2 (rows 21-40)
SELECT user_id, name, created_at
FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 20;
```

## The OFFSET Performance Problem

OFFSET N tells ClickHouse to scan and discard N rows before returning results. At large offsets this becomes slow:

```sql
-- Slow: scans 1,000,000 rows to return 20
SELECT * FROM events ORDER BY event_time DESC LIMIT 20 OFFSET 1000000;
```

## Keyset Pagination (Recommended for Large Tables)

Keyset pagination uses the last seen value as a cursor instead of an offset:

```sql
-- First page
SELECT event_id, user_id, event_time, amount
FROM events
ORDER BY event_time DESC, event_id DESC
LIMIT 20;

-- Next page (using last event_time and event_id from previous result)
SELECT event_id, user_id, event_time, amount
FROM events
WHERE (event_time, event_id) < ('2024-01-15 10:30:00', 98765)
ORDER BY event_time DESC, event_id DESC
LIMIT 20;
```

This is O(log n) regardless of which page you are on.

## Row Number Based Pagination

For arbitrary page jumps with windowing:

```sql
SELECT *
FROM (
    SELECT *,
        row_number() OVER (ORDER BY event_time DESC) AS rn
    FROM events
    WHERE event_time >= today() - 30
)
WHERE rn BETWEEN 41 AND 60;
```

## Getting Total Count Efficiently

Avoid a separate full scan for total count:

```sql
SELECT
    count() AS total_rows,
    uniq(user_id) AS unique_users
FROM events
WHERE event_time >= today() - 30;
```

Run this once and cache the result in your application layer.

## Stable Pagination with a Tie-Breaker

Always include a unique column as a tie-breaker to ensure stable page order:

```sql
SELECT order_id, customer_id, amount, created_at
FROM orders
WHERE (created_at, order_id) < ({last_created_at:DateTime}, {last_order_id:UInt64})
ORDER BY created_at DESC, order_id DESC
LIMIT 50;
```

## Paginating Aggregated Results

For paginating GROUP BY results:

```sql
SELECT
    category,
    sum(revenue) AS total_revenue,
    count() AS orders
FROM orders
GROUP BY category
ORDER BY total_revenue DESC
LIMIT 10 OFFSET {page_offset:UInt64};
```

## Summary

Use LIMIT/OFFSET for small tables and infrequent pagination. For large datasets and API endpoints, keyset pagination with `WHERE (ts, id) < (last_ts, last_id)` eliminates offset scans entirely. Always include a unique tie-breaker column in the ORDER BY to ensure stable, reproducible page ordering.
