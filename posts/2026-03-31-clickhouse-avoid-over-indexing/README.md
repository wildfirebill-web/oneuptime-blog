# Why You Should Avoid Over-Indexing in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Skip Index, Primary Index, Performance, Best Practice

Description: Explains why adding too many indexes to ClickHouse tables slows down inserts and merges, and how to choose indexes that actually benefit your queries.

---

## How Indexing Works in ClickHouse

ClickHouse has two indexing mechanisms:

1. **Primary (sparse) index** - defined by `ORDER BY`, stored in `primary.idx`, one entry per granule (~8192 rows)
2. **Data skipping indexes** - secondary indexes stored alongside data parts, allow skipping granules that cannot match a filter

Adding indexes is not free. Every `INSERT` and background merge must write and maintain index data. Too many skip indexes increase part write time, merge CPU cost, and disk usage.

## Primary Index: Keep It Simple

The primary index is the most powerful tool. The first few columns should cover your highest-selectivity filter/GROUP BY patterns.

```sql
-- Good: two-column primary key, selective for common queries
CREATE TABLE events (
  user_id    UInt64,
  event_time DateTime,
  event_type String
) ENGINE = MergeTree()
ORDER BY (user_id, event_time);

-- Over-engineered: adding low-cardinality column to primary key wastes space
CREATE TABLE events (
  user_id    UInt64,
  event_time DateTime,
  event_type String,
  country    String
) ENGINE = MergeTree()
ORDER BY (user_id, event_type, country, event_time);  -- usually too many
```

## Skip Indexes: Add Only When They Provide Measurable Benefit

A skip index helps when it can eliminate a significant portion of granules. If your column has low cardinality or data is not correlated with its storage order, the skip index helps little but still costs CPU on every insert.

```sql
-- This bloom filter index only helps if queries filter on session_id often
-- AND session_id values are not already correlated with ORDER BY
ALTER TABLE events ADD INDEX idx_session session_id TYPE bloom_filter GRANULARITY 4;
```

## Checking Whether a Skip Index Is Being Used

```sql
EXPLAIN indexes = 1
SELECT * FROM events
WHERE session_id = 'abc123'
  AND user_id = 42;
```

If `Granules: N/M` where N is close to M, the index is not filtering much - it is mostly overhead.

## Cost of Over-Indexing on Insert

```sql
-- Measure insert time with vs without index
-- Check system.query_log after inserts
SELECT
  query_duration_ms,
  written_rows,
  written_bytes
FROM system.query_log
WHERE query LIKE 'INSERT INTO events%'
  AND type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 10;
```

## When to Remove an Index

```sql
-- Remove a skip index that is not benefiting queries
ALTER TABLE events DROP INDEX idx_session;
```

Monitor query performance for a week after removing - if no regression, the index was wasted overhead.

## Summary

Over-indexing in ClickHouse increases insert latency, merge CPU usage, and disk consumption without proportional query benefit. The primary (sparse) index built from `ORDER BY` is sufficient for most workloads. Add skip indexes only after verifying with `EXPLAIN indexes = 1` that they actually eliminate granules. Remove indexes that do not improve query plans and measure the insert speed improvement.
