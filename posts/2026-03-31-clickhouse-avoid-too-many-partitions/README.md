# Why You Should Avoid Too Many Partitions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Partition, MergeTree, Performance, Best Practice

Description: Learn why over-partitioning tables in ClickHouse hurts performance and how to choose partition keys that balance manageability with query speed.

---

## Partitioning in ClickHouse

Partitions in ClickHouse organize data on disk into separate directories. They are useful for dropping old data efficiently and enabling partition pruning in queries. But partitions are not the primary indexing mechanism - that role belongs to the primary key (ORDER BY). Over-partitioning creates serious problems.

## What Happens with Too Many Partitions

Each partition maintains its own set of data parts. With many partitions:

- Background merges run per-partition, so merges cannot compact data across partitions
- Each INSERT that touches multiple partitions creates parts in each partition
- Query planning must scan partition metadata for every partition touched
- Memory and file descriptor usage increases with partition count

## Common Mistake: High-Cardinality Partition Key

```sql
-- WRONG: user_id has millions of distinct values
CREATE TABLE user_events (
  user_id    UInt64,
  event_type String,
  event_time DateTime
) ENGINE = MergeTree()
PARTITION BY user_id          -- creates millions of partitions!
ORDER BY (user_id, event_time);

-- WRONG: partitioning by timestamp with second precision
CREATE TABLE events (
  event_time DateTime,
  event_type String
) ENGINE = MergeTree()
PARTITION BY event_time       -- one partition per second!
ORDER BY event_time;
```

## Recommended Partition Key Patterns

```sql
-- GOOD: partition by month (creates 12 partitions per year)
CREATE TABLE events (
  event_time DateTime,
  event_type String,
  user_id    UInt64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);

-- GOOD: partition by day for high-volume tables with short retention
CREATE TABLE access_logs (
  log_time   DateTime,
  path       String,
  status     UInt16
) ENGINE = MergeTree()
PARTITION BY toDate(log_time)
ORDER BY (status, log_time);
```

## Checking Partition Count

```sql
SELECT
  table,
  count(DISTINCT partition) AS partition_count,
  count() AS part_count
FROM system.parts
WHERE active = 1
  AND database = currentDatabase()
GROUP BY table
ORDER BY partition_count DESC;
```

A healthy table typically has fewer than 1,000 partitions. Tens of thousands of partitions is a red flag.

## Dropping Partitions for Data Lifecycle

The main practical benefit of partitioning is fast data deletion:

```sql
-- Drop a specific month's data without touching the rest
ALTER TABLE events DROP PARTITION '202502';

-- Drop last month
ALTER TABLE events DROP PARTITION toYYYYMM(addMonths(today(), -1));
```

This works in milliseconds regardless of data volume - far faster than `DELETE`.

## Summary

Partition ClickHouse tables by time granularity (month or day) rather than by high-cardinality columns like user ID or precise timestamps. Keep partition count well under 1,000 per table. Use partitions for data lifecycle management (fast drops and TTL policies), not as a substitute for the primary key index.
