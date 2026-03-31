# How to Chain Multiple Materialized Views in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, Data Pipeline, Aggregation, Performance

Description: Learn how to chain multiple materialized views in ClickHouse to build multi-stage data pipelines that progressively aggregate raw event data.

---

## What is View Chaining

In ClickHouse, materialized views trigger when data is inserted into their source table. You can chain views by making one view write to an intermediate table that another view reads from. This creates a multi-stage aggregation pipeline.

## Stage 1: Raw Events Table

```sql
CREATE TABLE raw_events (
    event_time DateTime,
    user_id UInt64,
    event_type LowCardinality(String),
    value Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, event_time);
```

## Stage 2: Per-Minute Aggregates

Create an intermediate table and view:

```sql
CREATE TABLE events_per_minute (
    minute DateTime,
    event_type LowCardinality(String),
    event_count UInt64,
    total_value Float64,
    unique_users AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree()
ORDER BY (event_type, minute);

CREATE MATERIALIZED VIEW mv_events_per_minute
TO events_per_minute
AS SELECT
    toStartOfMinute(event_time) AS minute,
    event_type,
    count() AS event_count,
    sum(value) AS total_value,
    uniqState(user_id) AS unique_users
FROM raw_events
GROUP BY minute, event_type;
```

## Stage 3: Per-Hour Aggregates from the Intermediate Table

```sql
CREATE TABLE events_per_hour (
    hour DateTime,
    event_type LowCardinality(String),
    event_count UInt64,
    total_value Float64
) ENGINE = SummingMergeTree()
ORDER BY (event_type, hour);

CREATE MATERIALIZED VIEW mv_events_per_hour
TO events_per_hour
AS SELECT
    toStartOfHour(minute) AS hour,
    event_type,
    sum(event_count) AS event_count,
    sum(total_value) AS total_value
FROM events_per_minute
GROUP BY hour, event_type;
```

## Querying the Chain

Query raw data for recent events:

```sql
SELECT event_type, count() FROM raw_events
WHERE event_time >= now() - INTERVAL 5 MINUTE
GROUP BY event_type;
```

Query the hourly rollup for historical trends:

```sql
SELECT hour, event_type, event_count, total_value
FROM events_per_hour
WHERE hour >= now() - INTERVAL 7 DAY
ORDER BY hour DESC;
```

## Important Considerations

- Each view in the chain adds write amplification. Keep chains to 2-3 stages.
- Views only process new inserts - they do not reprocess existing data automatically.
- Use `POPULATE` carefully when creating new views on existing tables.
- Test your chain with small inserts before going to production.

```sql
-- Verify chain is working
INSERT INTO raw_events VALUES (now(), 42, 'click', 1.0);
SELECT * FROM events_per_minute WHERE minute >= now() - INTERVAL 1 MINUTE;
SELECT * FROM events_per_hour WHERE hour >= now() - INTERVAL 1 HOUR;
```

## Summary

Chaining materialized views in ClickHouse lets you build efficient multi-tier aggregation pipelines. Raw events feed minute-level summaries, which in turn feed hourly rollups - each tier serving different query patterns at different time granularities without reprocessing raw data.
