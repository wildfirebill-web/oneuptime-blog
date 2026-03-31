# How to Choose the Right ORDER BY Key in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ORDER BY, Primary Key, Schema Design, MergeTree, Performance

Description: Learn how to choose the optimal ORDER BY key in ClickHouse MergeTree tables to maximize query performance and compression.

---

The `ORDER BY` key in ClickHouse MergeTree is the primary sort key - it determines both query performance (via primary index) and data compression. Choosing it correctly is the most impactful schema design decision.

## How ORDER BY Key Works

ClickHouse stores data sorted by the ORDER BY columns. The primary index stores the values of ORDER BY columns at every 8192nd row (one "mark"). Queries that filter on ORDER BY columns skip marks and therefore disk reads:

```sql
CREATE TABLE events (
  region LowCardinality(String),
  event_time DateTime,
  user_id UInt64,
  event_type LowCardinality(String),
  amount Decimal64(2)
) ENGINE = MergeTree()
ORDER BY (region, event_time);
-- Queries filtering on region and/or event_time will be fast
```

## Put High-Cardinality Equality Filters Last

For compound keys, put low-cardinality columns first and high-cardinality columns later:

```sql
-- Good: region (50 values) before user_id (millions)
ORDER BY (region, event_type, event_time, user_id)

-- Queries like this benefit:
-- WHERE region = 'US' AND event_type = 'purchase'
```

The primary index prunes most efficiently when you filter on the leftmost columns.

## Always Include a Timestamp

For time-series data, always include a datetime column in the ORDER BY. This enables:
- Time range queries to skip most data
- TTL to work efficiently
- Partition pruning to work with ORDER BY together

```sql
ORDER BY (region, toStartOfHour(event_time), user_id)
-- Or
ORDER BY (region, event_time)
```

## Choose ORDER BY for Your Most Common Queries

Analyze your query patterns and build the ORDER BY around the most frequent WHERE clauses:

```sql
-- If most queries are: WHERE tenant_id = ? AND created_at BETWEEN ? AND ?
ORDER BY (tenant_id, created_at)

-- If most queries are: WHERE service_name = ? AND level = ? AND ts >= ?
ORDER BY (service_name, level, ts)
```

## Avoid High-Cardinality Columns as First Key

A UUID or user_id as the first ORDER BY column means every query without that filter does a full scan:

```sql
-- BAD: UUID first - no prefix pruning possible
ORDER BY (request_id, event_time, region)

-- GOOD: low-cardinality filter columns first
ORDER BY (region, event_time, request_id)
```

## Compression Benefits

Data sorted by low-cardinality columns compresses better. If region has 50 values, 8192 consecutive rows likely have the same region and compress to near zero:

```sql
-- Check compression ratio per column
SELECT
  column,
  formatReadableSize(sum(data_compressed_bytes)) AS compressed,
  formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
  round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
WHERE table = 'events'
GROUP BY column
ORDER BY ratio ASC;
```

## Use Projections for Secondary Access Patterns

If you have two different common query patterns, use a projection for the secondary one:

```sql
ALTER TABLE events
  ADD PROJECTION events_by_user (
    SELECT * ORDER BY user_id, event_time
  );

ALTER TABLE events MATERIALIZE PROJECTION events_by_user;
```

## Summary

The `ORDER BY` key in ClickHouse MergeTree should prioritize columns used in WHERE clause equality filters, ordered from lowest cardinality to highest, and must include a timestamp for time-series data. The leftmost columns are the most important for query pruning. Use projections to support secondary access patterns without changing the primary ORDER BY.
