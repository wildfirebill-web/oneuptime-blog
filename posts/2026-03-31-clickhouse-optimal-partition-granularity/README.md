# How to Choose Optimal Partition Granularity in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Partition, Performance, Schema Design, MergeTree, Storage

Description: Learn how to choose the right partition granularity in ClickHouse to balance query performance, merge efficiency, and storage overhead.

---

Partitioning in ClickHouse splits table data into separate directories based on a key. Choosing the right granularity is critical - too coarse and query pruning is ineffective; too fine and you create too many parts, causing merge overhead.

## How Partitioning Works

ClickHouse stores data in parts within partitions. When you filter by the partition key, ClickHouse reads only the relevant partitions (partition pruning). Each INSERT creates a new part, which background merges combine into fewer larger parts.

## Granularity Options

```sql
-- Monthly (most common for time-series)
PARTITION BY toYYYYMM(event_time)

-- Daily (for high-volume tables with day-level retention)
PARTITION BY toDate(event_time)

-- Yearly (for cold/archival data)
PARTITION BY toYear(event_time)

-- Weekly
PARTITION BY toMonday(event_time)

-- Hourly (rarely needed, creates too many parts)
PARTITION BY toStartOfHour(event_time)
```

## The Right Granularity by Data Volume

Match partition granularity to your query patterns and data volume:

| Daily Ingestion | Recommended Granularity |
| --- | --- |
| < 1 GB | Monthly or yearly |
| 1 GB - 100 GB | Monthly |
| 100 GB - 1 TB | Daily |
| > 1 TB | Daily with careful merge tuning |

## Too-Fine Granularity: The "Too Many Parts" Problem

If partitions are too small:

```sql
-- Check if you have too many parts
SELECT
    partition,
    count() AS part_count,
    sum(rows) AS total_rows,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE table = 'your_table' AND active = 1
GROUP BY partition
HAVING part_count > 100
ORDER BY part_count DESC;
```

Too many parts cause:
- Slow inserts (too many file descriptors)
- Slow queries (more files to open)
- Excessive merge background work

## Too-Coarse Granularity: No Pruning Benefit

If a single partition contains all your data, partition pruning provides no benefit:

```sql
-- Check partition size distribution
SELECT
    partition,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    sum(rows) AS rows
FROM system.parts
WHERE table = 'events' AND active = 1
GROUP BY partition
ORDER BY rows DESC;
```

If one partition has billions of rows, consider finer granularity.

## Combining Partition Key with Primary Key

Partitioning and primary key work together for optimal pruning:

```sql
-- Partition by month, primary key filters within partition
CREATE TABLE events (
    event_time DateTime,
    event_type LowCardinality(String),
    user_id UInt64,
    value Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, user_id, event_time);

-- This query uses both partition pruning AND primary key range scan
SELECT sum(value)
FROM events
WHERE event_time >= '2026-01-01' AND event_time < '2026-02-01'
  AND event_type = 'purchase';
```

## Changing Partition Granularity

You cannot change partitioning on an existing table. Create a new table and migrate:

```sql
CREATE TABLE events_new AS events
ENGINE = MergeTree()
PARTITION BY toDate(event_time)  -- Changed from monthly to daily
ORDER BY (event_type, user_id, event_time);

INSERT INTO events_new SELECT * FROM events;
RENAME TABLE events TO events_old, events_new TO events;
```

## Monitoring Merge Health

Check that merges are keeping part counts under control:

```sql
SELECT
    table,
    count() AS active_parts,
    sum(rows) AS total_rows,
    formatReadableSize(sum(bytes_on_disk)) AS total_size
FROM system.parts
WHERE active = 1
GROUP BY table
ORDER BY active_parts DESC
LIMIT 20;
```

## Summary

Choose partition granularity based on data volume and query patterns: monthly for most time-series tables (1 GB - 100 GB/day), daily for high-volume tables, and yearly for archival data. Avoid hourly partitioning in production as it creates too many parts. Use EXPLAIN ESTIMATE to verify partition pruning is working before deploying schema changes.
