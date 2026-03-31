# How to Set Up Your First ClickHouse Materialized View

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, Query Optimization, Aggregation, Performance

Description: Learn how to create your first ClickHouse materialized view to pre-aggregate data on insert and dramatically speed up dashboard queries.

---

Materialized views in ClickHouse are one of the most powerful features for query acceleration. Unlike traditional databases where materialized views are refreshed periodically, ClickHouse materialized views are triggered on every INSERT and maintain running aggregates in real time.

## How ClickHouse Materialized Views Work

When you insert data into a source table, ClickHouse automatically runs the materialized view query on the new block and inserts the result into a target table. The target table accumulates partial aggregations, and ClickHouse merges them in the background.

## Source Table Setup

```sql
CREATE TABLE page_views (
    event_time DateTime,
    page_path String,
    user_id UInt32,
    country LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (page_path, event_time);
```

## Creating the Target Table

The target table must use an aggregating engine that stores partial states:

```sql
CREATE TABLE page_views_hourly (
    hour DateTime,
    page_path String,
    country LowCardinality(String),
    views AggregateFunction(count, UInt32),
    unique_users AggregateFunction(uniq, UInt32)
) ENGINE = AggregatingMergeTree()
ORDER BY (hour, page_path, country);
```

## Creating the Materialized View

```sql
CREATE MATERIALIZED VIEW page_views_mv
TO page_views_hourly
AS
SELECT
    toStartOfHour(event_time) AS hour,
    page_path,
    country,
    countState(user_id) AS views,
    uniqState(user_id) AS unique_users
FROM page_views
GROUP BY hour, page_path, country;
```

## Querying the Materialized View

Use the `Merge` suffix functions to combine partial states:

```sql
SELECT
    hour,
    page_path,
    country,
    countMerge(views) AS total_views,
    uniqMerge(unique_users) AS unique_users
FROM page_views_hourly
WHERE hour >= now() - INTERVAL 24 HOUR
GROUP BY hour, page_path, country
ORDER BY hour, total_views DESC;
```

## Populating Existing Data

By default, the materialized view only captures future inserts. To backfill existing data:

```sql
INSERT INTO page_views_hourly
SELECT
    toStartOfHour(event_time) AS hour,
    page_path,
    country,
    countState(user_id) AS views,
    uniqState(user_id) AS unique_users
FROM page_views
GROUP BY hour, page_path, country;
```

## Common Pitfalls

- Do not use `count()` in the target table - use `countState()` in the view and `countMerge()` when reading.
- If you alter the source table schema, you may need to recreate the materialized view.
- The materialized view processes each INSERT block independently, so avoid tiny single-row inserts.

## Summary

ClickHouse materialized views pre-aggregate data on insert using `AggregatingMergeTree` as the storage engine. By using aggregate state functions, you can maintain accurate running totals of counts, sums, and approximate unique counts that are merged at query time, turning slow full-table scans into instant lookups.
