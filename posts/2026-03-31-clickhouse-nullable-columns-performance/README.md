# How to Avoid Performance Penalties of Nullable Columns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Nullable, Performance, Column Design, MergeTree, Schema

Description: Understand why Nullable columns slow down ClickHouse queries and learn practical strategies to avoid or minimize their performance impact.

---

## The Cost of Nullable Columns

In ClickHouse, wrapping a column in `Nullable(T)` stores an additional bitmask file alongside the data to track which values are NULL. This has several consequences:

- Extra disk reads for every query touching that column
- Inability to use certain indexes and optimizations
- Higher memory usage during aggregation
- Functions that operate on Nullable types are generally slower than their non-nullable counterparts

## How ClickHouse Stores Nullable Columns

For a column declared as `Nullable(UInt64)`, ClickHouse creates two files per part:

```text
column_name.bin         -- actual values
column_name.null.bin    -- null bitmask
```

Every read must load both files. For high-frequency query patterns on large datasets, this doubles I/O for that column.

## Benchmarking the Difference

Consider a 100 million row table. A simple aggregation shows measurable overhead:

```sql
-- Nullable version
SELECT sum(revenue) FROM events;   -- ~0.8s

-- Non-nullable with sentinel value
SELECT sumIf(revenue, revenue != -1) FROM events;  -- ~0.4s
```

The exact difference varies by hardware and data distribution, but the penalty is real.

## Strategy 1: Use Sentinel Values

Replace NULL with a sentinel that has no valid business meaning:

```sql
CREATE TABLE events (
    event_id    UInt64,
    user_id     UInt64,
    revenue     Float64 DEFAULT 0.0,   -- 0 means no revenue, not NULL
    status_code Int16   DEFAULT -1     -- -1 means unknown
) ENGINE = MergeTree()
ORDER BY (event_id);
```

Document the sentinel values clearly in your schema or a data dictionary.

## Strategy 2: Separate Presence from Value

Split a nullable column into a presence flag and a non-nullable value column:

```sql
CREATE TABLE measurements (
    sensor_id     UInt32,
    ts            DateTime,
    has_reading   UInt8,       -- 0 or 1
    reading_value Float64      -- meaningful only when has_reading = 1
) ENGINE = MergeTree()
ORDER BY (sensor_id, ts);
```

This avoids Nullable entirely while preserving the semantic distinction.

## Strategy 3: Use LowCardinality Instead of Nullable String

For string columns with a small number of distinct values including potential NULLs, use `LowCardinality(String)` with an empty string as the null marker:

```sql
CREATE TABLE users (
    user_id  UInt64,
    tier     LowCardinality(String) DEFAULT ''  -- empty = no tier assigned
) ENGINE = MergeTree()
ORDER BY user_id;
```

`LowCardinality` is highly optimized and avoids the Nullable overhead.

## When Nullable Is Acceptable

Not all Nullable usage needs to be eliminated. It is reasonable when:

- The column is rarely queried in aggregations
- Data arrives from external sources where NULL has a distinct semantic meaning
- The table is small or infrequently accessed

## Checking Your Schema for Nullable Columns

```sql
SELECT
    name,
    type
FROM system.columns
WHERE database = 'mydb'
  AND table = 'events'
  AND type LIKE 'Nullable%';
```

## Summary

Nullable columns in ClickHouse carry a real I/O and CPU cost due to the extra null bitmask. For performance-critical tables, prefer sentinel values, presence flags, or `LowCardinality(String)` over `Nullable(T)`. Reserve Nullable for columns where NULL has distinct semantic meaning and is not on a hot query path.
