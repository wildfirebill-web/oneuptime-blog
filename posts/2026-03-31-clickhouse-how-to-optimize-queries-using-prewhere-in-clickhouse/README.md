# How to Optimize Queries Using PREWHERE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, PREWHERE, Query Optimization, Performance, MergeTree

Description: Learn how to use PREWHERE in ClickHouse to apply selective filters before reading all columns, reducing I/O and improving query performance on large tables.

---

## What Is PREWHERE

`PREWHERE` is a ClickHouse-specific optimization clause for MergeTree tables. When you specify a condition in `PREWHERE`, ClickHouse reads only the columns referenced in that condition first, evaluates the filter, and then reads the remaining columns only for rows that pass the filter.

This reduces I/O dramatically when the PREWHERE condition is highly selective - only a small fraction of rows pass.

## Basic PREWHERE Syntax

```sql
SELECT
    user_id,
    event_type,
    payload
FROM events
PREWHERE event_date >= today() - 7
WHERE event_type = 'purchase';
```

In this query, ClickHouse reads `event_date` first, filters to rows from the last 7 days, then reads `user_id`, `event_type`, and `payload` only for those rows.

## Automatic PREWHERE Optimization

ClickHouse automatically applies PREWHERE optimization for most WHERE conditions - you don't always need to write it explicitly. The optimizer moves selective, low-cost conditions to PREWHERE automatically.

Check if it's being applied:

```sql
EXPLAIN
SELECT count()
FROM events
WHERE event_date >= today() - 7
  AND event_type = 'purchase';
```

## When to Explicitly Use PREWHERE

Use explicit PREWHERE when you have a highly selective condition on a small column and the automatic optimizer is not choosing it:

```sql
SELECT
    user_id,
    session_id,
    payload,      -- Large string column
    metadata      -- Another large column
FROM events
PREWHERE status_code = 500  -- Small integer, very selective
WHERE error_message LIKE '%timeout%';
```

## Multiple Conditions with PREWHERE

You can combine multiple conditions in PREWHERE:

```sql
SELECT
    user_id,
    amount,
    cart_data  -- Large JSON column
FROM orders
PREWHERE
    order_date >= today() - 30
    AND status = 'completed'
WHERE amount > 1000;
```

## PREWHERE with Partition Columns

PREWHERE works especially well with partition columns since reading one column per granule is cheap:

```sql
CREATE TABLE events (
    event_date  Date,
    user_id     UInt64,
    event_type  String,
    payload     String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (user_id, event_date);

SELECT user_id, payload
FROM events
PREWHERE event_date >= today() - 90
WHERE event_type = 'error';
```

## What Cannot Be Used in PREWHERE

- Columns included in `ALIAS` definitions
- Columns used in sampling expressions
- `arrayJoin` expressions
- Subqueries

## Measuring PREWHERE Effectiveness

```sql
-- Without PREWHERE
SELECT count()
FROM large_events
WHERE event_date = today()
  AND event_type = 'purchase';

-- With explicit PREWHERE
SELECT count()
FROM large_events
PREWHERE event_date = today()
WHERE event_type = 'purchase';

-- Compare read_rows and read_bytes in system.query_log
SELECT
    query_id,
    read_rows,
    read_bytes,
    result_rows,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date = today()
ORDER BY event_time DESC
LIMIT 10;
```

## Disabling Automatic PREWHERE

To disable automatic PREWHERE optimization (useful for benchmarking):

```sql
SELECT count()
FROM events
WHERE event_date = today()
SETTINGS optimize_move_to_prewhere = 0;
```

## Combining PREWHERE with Skip Indexes

PREWHERE evaluates after skip index pruning, so the combination is powerful:

```sql
SELECT user_id, payload
FROM events
PREWHERE event_date >= today() - 7  -- Skip index on date
WHERE session_id = 'abc123';        -- bloom_filter skip index on session_id
```

## Summary

PREWHERE is ClickHouse's mechanism for reading the minimum data necessary to evaluate selective filters before loading expensive columns. While ClickHouse applies it automatically in most cases via `optimize_move_to_prewhere`, explicit PREWHERE control is valuable for large-column tables where you need fine-grained optimization. Combine it with skip indexes for maximum I/O reduction on large MergeTree tables.
