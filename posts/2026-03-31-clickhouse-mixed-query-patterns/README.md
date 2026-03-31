# How to Design Schemas for Mixed Query Patterns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Schema Design, Projection, Mixed Workload, MergeTree, Performance

Description: Design ClickHouse schemas that perform well for multiple different query patterns using projections, materialized views, and tiered storage.

---

Most production ClickHouse deployments have mixed query patterns: time-range aggregations, point lookups by user ID, and funnel analyses. A single ORDER BY key cannot optimize all of them. This guide shows how to handle mixed workloads.

## Understand the Trade-off

The ORDER BY key optimizes queries that filter on its leftmost columns. For a table ordered by `(service, timestamp)`:

```sql
-- FAST: uses primary index
SELECT count() FROM logs WHERE service = 'api' AND timestamp >= now() - INTERVAL 1 DAY;

-- SLOW: no prefix match for user_id
SELECT count() FROM logs WHERE user_id = 42;
```

## Use Projections for Secondary Access Patterns

Projections store a copy of data in a different order, allowing multiple primary keys:

```sql
CREATE TABLE events (
  event_time DateTime,
  user_id UInt64,
  region LowCardinality(String),
  event_type LowCardinality(String),
  amount Decimal64(2)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (region, event_time);

-- Add projection for user-centric queries
ALTER TABLE events
  ADD PROJECTION events_by_user (
    SELECT * ORDER BY (user_id, event_time)
  );

ALTER TABLE events MATERIALIZE PROJECTION events_by_user;
```

Now both patterns are fast:

```sql
-- Uses primary ORDER BY
SELECT sum(amount) FROM events WHERE region = 'US' AND event_time >= today() - 7;

-- Uses projection
SELECT sum(amount) FROM events WHERE user_id = 42;
```

## Separate Tables for Distinct Workloads

For very different workloads, separate tables can be cleaner than projections:

```sql
-- Raw events: time-series queries
CREATE TABLE raw_events (
  event_time DateTime,
  user_id UInt64,
  ...
) ENGINE = MergeTree()
ORDER BY (event_time);

-- User activity summary: user-centric queries
CREATE TABLE user_daily_summary (
  event_date Date,
  user_id UInt64,
  event_count UInt32,
  total_amount Decimal64(2)
) ENGINE = SummingMergeTree()
ORDER BY (user_id, event_date);
```

## Use Materialized Views for Aggregation

Pre-aggregate for dashboard query patterns:

```sql
CREATE MATERIALIZED VIEW hourly_metrics
ENGINE = SummingMergeTree()
ORDER BY (region, event_type, event_hour)
AS
SELECT
  region,
  event_type,
  toStartOfHour(event_time) AS event_hour,
  count() AS events,
  sum(amount) AS revenue
FROM events
GROUP BY region, event_type, event_hour;
```

## Add Skip Indexes for Lookup Queries

Bloom filter indexes enable efficient point lookups on non-primary-key columns:

```sql
ALTER TABLE events
  ADD INDEX user_id_bloom user_id TYPE bloom_filter(0.01) GRANULARITY 1;
ALTER TABLE events MATERIALIZE INDEX user_id_bloom;

-- Now efficient even without projection
SELECT * FROM events WHERE user_id = 42 LIMIT 100;
```

## Monitor Query Patterns

Identify which queries are slow to guide your schema:

```sql
SELECT
  normalized_query_hash,
  count() AS executions,
  avg(query_duration_ms) AS avg_ms,
  max(query_duration_ms) AS max_ms,
  any(query) AS sample_query
FROM system.query_log
WHERE type = 'QueryFinish' AND event_date >= today() - 7
GROUP BY normalized_query_hash
ORDER BY avg_ms * executions DESC
LIMIT 20;
```

## Summary

Mixed query pattern schemas in ClickHouse use the ORDER BY key for the most frequent query pattern, projections for secondary sort orders, materialized views for pre-aggregated access patterns, and bloom filter skip indexes for point lookups. Monitor `system.query_log` to identify high-impact slow queries and prioritize which secondary pattern to optimize first.
