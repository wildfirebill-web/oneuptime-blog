# How to Configure ClickHouse for Interactive Query Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Interactive Query, Latency, Cache, Configuration, Performance

Description: Tune ClickHouse for sub-second interactive query latency using query cache, pre-aggregations, projection indexes, and connection pool settings.

---

## What Makes a Query Interactive?

Interactive queries need results in under a second - or ideally under 100ms. Users don't wait for scans of billions of rows. ClickHouse can achieve interactive latency through smart schema design, caching, and pre-aggregation.

## Enable Query Result Cache

ClickHouse 23.x+ supports a query result cache:

```sql
SET use_query_cache = 1;
SET query_cache_ttl = 60;        -- cache for 60 seconds
SET query_cache_min_query_duration = 100;  -- only cache queries > 100ms
```

Repeated identical queries return instantly from cache:

```sql
SELECT count(), sum(revenue)
FROM orders
WHERE order_date = today()
SETTINGS use_query_cache = 1, query_cache_ttl = 30;
```

## Pre-Aggregate with Materialized Views

Move aggregation work from query time to insert time:

```sql
CREATE MATERIALIZED VIEW hourly_revenue_mv
ENGINE = SummingMergeTree()
ORDER BY (hour, product_category)
AS SELECT
    toStartOfHour(ts) AS hour,
    product_category,
    sum(revenue)      AS total_revenue,
    count()           AS order_count
FROM orders;
```

Queries against `hourly_revenue_mv` scan far fewer rows than the raw table.

## Use Projections for Common Filters

Projections store pre-sorted subsets of data:

```sql
ALTER TABLE orders
ADD PROJECTION orders_by_user
(
    SELECT user_id, order_date, revenue
    ORDER BY (user_id, order_date)
);

ALTER TABLE orders MATERIALIZE PROJECTION orders_by_user;
```

Queries filtering by `user_id` now use the projection's sort order automatically.

## Limit Threads for Fair Concurrency

Interactive workloads need many queries running simultaneously, each with fewer threads:

```sql
SET max_threads = 8;
SET max_memory_usage = 4000000000;
SET max_execution_time = 10;
```

## Mark Cache and Uncompressed Cache

Tune in-memory caches for hot data:

```xml
<!-- config.xml -->
<mark_cache_size>5368709120</mark_cache_size>        <!-- 5 GB mark cache -->
<uncompressed_cache_size>8589934592</uncompressed_cache_size>  <!-- 8 GB uncompressed -->
```

Mark cache stores index marks (position pointers), uncompressed cache stores decompressed column chunks. Both dramatically speed up repeated scans of the same data.

## Optimize Sort Keys for Common Queries

The sort key determines which queries benefit from index pruning. Align it with your most frequent filter:

```sql
ORDER BY (tenant_id, event_date, event_type)
```

Queries filtering by `tenant_id` alone will skip most of the table.

## Summary

Configure ClickHouse for interactive queries by enabling query result cache, pre-aggregating with materialized views, using projections for alternate sort orders, tuning mark and uncompressed caches, and limiting per-query thread count for fairness. The biggest wins come from schema design - getting the sort key right reduces scan cost before any tuning is needed.
