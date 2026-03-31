# How to Choose Optimal Partition Granularity in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Partition, Granularity, MergeTree, Performance, Schema Design

Description: Choose the right partition granularity in ClickHouse to balance query pruning efficiency, merge performance, and part file count.

---

Partitioning in ClickHouse is a physical data organization strategy. Choosing the wrong partition granularity leads to either too many parts (slow merges) or too few partitions (no pruning benefit). This guide helps you choose the right granularity.

## How Partitioning Works

ClickHouse stores each partition in separate directories. When a query includes a filter on the partition key, ClickHouse skips entire partitions:

```sql
CREATE TABLE events (
  event_time DateTime,
  region LowCardinality(String),
  user_id UInt64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)  -- Monthly partitions
ORDER BY (region, event_time);
```

## When to Use Monthly Partitions

Monthly is the most common choice for time-series data where:
- You query one or a few months at a time
- You need to drop old data by month (TTL or manual)
- Total data size per month is 1GB-100GB

```sql
PARTITION BY toYYYYMM(event_time)  -- 12 partitions/year
```

## When to Use Daily Partitions

Use daily partitions when:
- Queries almost always filter to 1-7 days
- You need sub-monthly data lifecycle management
- Each day's data exceeds 100MB

```sql
PARTITION BY toDate(event_time)  -- 365 partitions/year
```

Warning: Avoid daily partitions for tables with < 10MB/day - the overhead from many small parts outweighs the pruning benefit.

## When to Use Weekly or Yearly Partitions

```sql
-- Weekly: good for medium-volume tables with week-scoped queries
PARTITION BY toMonday(event_time)

-- Yearly: good for cold archive data with full-year queries
PARTITION BY toYear(event_time)
```

## Avoid Over-Partitioning

Too many partitions cause problems:

```sql
-- BAD: hourly partitions create 8760 partitions/year
PARTITION BY toStartOfHour(event_time)

-- BAD: composite partitions with high cardinality create too many parts
PARTITION BY (region, toDate(event_time))
-- Creates partitions: 50 regions * 365 days = 18,250 partitions/year
```

Check current part count:

```sql
SELECT
  partition,
  count() AS part_count,
  sum(rows) AS rows,
  formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE table = 'events' AND active
GROUP BY partition
ORDER BY partition;
```

If any partition has > 300 active parts, consider reducing granularity.

## Combine Partitioning with TTL

Partitioning enables efficient TTL-based data lifecycle management:

```sql
CREATE TABLE events (
  event_time DateTime,
  ...
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY event_time
TTL event_time + INTERVAL 12 MONTH DELETE;
```

TTL deletion removes whole partitions efficiently.

## Check Partition Pruning

Verify that partitions are pruned for your queries:

```sql
EXPLAIN PLAN
SELECT count() FROM events
WHERE event_time >= '2025-06-01' AND event_time < '2025-07-01';
-- Should show: Partitions: 1/12 (only June partition)
```

## Summary

Choose monthly partitions for most time-series workloads, daily partitions for high-volume tables with day-scoped queries, and avoid granularity finer than daily unless each partition holds substantial data. Always verify partition pruning with `EXPLAIN PLAN` and monitor part counts in `system.parts` to detect over-partitioning.
