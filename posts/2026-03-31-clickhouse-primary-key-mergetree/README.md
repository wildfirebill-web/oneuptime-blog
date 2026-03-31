# How to Use Primary Key Index in ClickHouse MergeTree

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Primary Key, MergeTree, Index, Query Optimization

Description: Learn how ClickHouse MergeTree primary key indexes work, how to design them for fast queries, and how to verify index usage with EXPLAIN.

---

## How MergeTree Primary Keys Work

Unlike RDBMS primary keys, ClickHouse's MergeTree primary key is a **sparse index** stored in `primary.idx`. It does not enforce uniqueness - it only helps skip data granules that don't match query filters.

By default, one index entry is created every 8192 rows (one granule). ClickHouse reads only the granules that could contain matching rows.

## Define a Primary Key

```sql
CREATE TABLE events (
    user_id UInt32,
    ts DateTime,
    event_type LowCardinality(String),
    value Float64
) ENGINE = MergeTree()
ORDER BY (user_id, ts)
PRIMARY KEY (user_id, ts);
```

If `PRIMARY KEY` is omitted, it defaults to the `ORDER BY` key.

## Primary Key vs ORDER BY Key

You can have a subset primary key:

```sql
CREATE TABLE events (
    user_id UInt32,
    ts DateTime,
    event_type String,
    value Float64
) ENGINE = MergeTree()
ORDER BY (user_id, ts, event_type)
PRIMARY KEY (user_id, ts);
```

The table is sorted by all three columns, but the index only covers the first two - balancing index size and selectivity.

## How Index Skipping Works

```sql
-- This query uses the primary index to skip granules
SELECT sum(value)
FROM events
WHERE user_id = 42 AND ts >= '2026-01-01' AND ts < '2026-02-01';
```

ClickHouse finds the range of granules containing `user_id = 42` and skips all others.

## Verify Index Usage with EXPLAIN

```sql
EXPLAIN indexes = 1
SELECT sum(value)
FROM events
WHERE user_id = 42 AND ts >= '2026-01-01';
```

Look for `Parts: N/M` where N is parts read and M is total. Fewer parts read = better index usage.

## Index Granularity Setting

```sql
CREATE TABLE events (
    user_id UInt32,
    ts DateTime,
    value Float64
) ENGINE = MergeTree()
ORDER BY (user_id, ts)
SETTINGS index_granularity = 8192;  -- default
```

Lower granularity = finer index but more index entries and memory. Higher granularity = coarser but smaller index.

## Check Index Size

```sql
SELECT
    formatReadableSize(primary_key_bytes_in_memory) AS pk_size
FROM system.parts
WHERE table = 'events' AND active = 1
ORDER BY part_name
LIMIT 10;
```

## Query That Does NOT Use the Index

```sql
-- ts is not the first key column - full scan within user_id granules
SELECT sum(value)
FROM events
WHERE ts >= '2026-01-01';
```

If `ts` is not the leading key, ClickHouse cannot effectively skip granules by `ts` alone.

## Summary

ClickHouse MergeTree primary keys are sparse indexes that skip data granules during query execution. Design your `ORDER BY` key with your most common filter columns first. Use `EXPLAIN indexes = 1` to verify the index is being used, and check `system.parts` to monitor index size and effectiveness.
