# How to Design Compound Primary Keys in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Primary Key, MergeTree, Index, Query Optimization

Description: Learn how to design effective compound primary keys in ClickHouse MergeTree tables to maximize granule pruning and query performance for your access patterns.

---

In ClickHouse, the primary key defines the sort order of data within MergeTree parts and is used to build a sparse index for granule-level pruning. Choosing the right compound key is one of the most impactful decisions for query performance.

## How the Primary Key Index Works

ClickHouse stores data sorted by the primary key and writes one index mark per granule (default 8192 rows). At query time, it uses a binary search over the sparse index to skip granules that cannot contain matching rows. The more selective your filter, the more granules are skipped.

## Basic Compound Key

```sql
CREATE TABLE events (
    user_id    UInt64,
    event_date Date,
    event_type String,
    value      Float64
) ENGINE = MergeTree()
ORDER BY (user_id, event_date, event_type);
```

The leftmost column in the key is the most important for pruning. Queries filtering on `user_id` alone skip irrelevant user ranges. Queries on `event_date` alone benefit less because `event_date` is the second column.

## Rule: Put the Most Selective Equality Column First

For queries like:
```sql
SELECT * FROM events WHERE user_id = 42 AND event_date = '2024-01-15'
```

Put `user_id` first because it narrows the range most:

```sql
ORDER BY (user_id, event_date)
-- Good: binary search on user_id first, then event_date within that range
```

Not:
```sql
ORDER BY (event_date, user_id)
-- Worse: has to scan all users for a given date
```

## When to Put a Low-Cardinality Column First

For time-series dashboards with queries like:
```sql
SELECT sum(value) FROM metrics WHERE service = 'api' AND ts >= now() - INTERVAL 1 HOUR
```

Put the low-cardinality filter first:
```sql
ORDER BY (service, ts)
```

This co-locates all data for a service together, so the time range scan is contiguous on disk.

## Cardinality vs Query Pattern Trade-off

```text
Access Pattern                       Recommended Key
--------------------                 ---------------
Lookup by user, then time            (user_id, event_date)
Dashboard by service, then time      (service, ts)
Multi-tenant analytics               (tenant_id, metric, ts)
Log analysis by level, then time     (level, log_time)
```

## Skipping Columns in the Middle

ClickHouse can prune on any prefix of the key. Filtering on the 3rd column without filtering on the 1st and 2nd provides no index benefit - those granules are scanned fully.

```sql
-- Benefits from (user_id, event_date, event_type) index:
WHERE user_id = 42                        -- prefix match, good pruning
WHERE user_id = 42 AND event_date = today() -- prefix match, better pruning

-- No index benefit:
WHERE event_type = 'click'                -- not a prefix, full scan
```

## Checking Granule Pruning with EXPLAIN

```sql
EXPLAIN indexes=1
SELECT count() FROM events WHERE user_id = 42 AND event_date = '2024-01-15'
```

Look for `Granules: X/Y` to see how many granules are skipped.

## Summary

Design compound primary keys in ClickHouse by placing the columns most commonly used in equality filters leftmost. For time-series data, place the dimension (service, tenant, user) before the timestamp. Verify your design with `EXPLAIN indexes=1` to confirm granule pruning is working for your most frequent query patterns.
