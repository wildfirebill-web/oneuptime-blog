# How to Use Materialized Views to Pre-Compute Aggregations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, Aggregation, Performance, AggregatingMergeTree

Description: Use ClickHouse materialized views with AggregatingMergeTree to pre-compute aggregations and serve dashboard queries in milliseconds.

---

Materialized views in ClickHouse are triggered on INSERT and store pre-computed results. Combined with `AggregatingMergeTree`, they maintain running aggregates that can be queried in milliseconds regardless of raw data volume.

## Create the Source Table

```sql
CREATE TABLE events (
  event_time DateTime,
  user_id UInt64,
  event_type String,
  revenue Decimal64(2)
) ENGINE = MergeTree()
ORDER BY (event_type, event_time);
```

## Create the Aggregating Table

Use `AggregatingMergeTree` with aggregate state columns:

```sql
CREATE TABLE events_daily_agg (
  event_date Date,
  event_type String,
  event_count AggregateFunction(count),
  total_revenue AggregateFunction(sum, Decimal64(2)),
  unique_users AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree()
ORDER BY (event_type, event_date);
```

## Create the Materialized View

The view triggers on every INSERT to `events` and writes to `events_daily_agg`:

```sql
CREATE MATERIALIZED VIEW events_daily_mv
TO events_daily_agg
AS
SELECT
  toDate(event_time) AS event_date,
  event_type,
  countState() AS event_count,
  sumState(revenue) AS total_revenue,
  uniqState(user_id) AS unique_users
FROM events
GROUP BY event_date, event_type;
```

## Query the Aggregated Data

Use the `-Merge` combinators to finalize the aggregate states:

```sql
SELECT
  event_date,
  event_type,
  countMerge(event_count) AS events,
  sumMerge(total_revenue) AS revenue,
  uniqMerge(unique_users) AS users
FROM events_daily_agg
WHERE event_date >= today() - 30
GROUP BY event_date, event_type
ORDER BY event_date, event_type;
```

## Backfill Existing Data

If you create the view after data already exists in the source table, backfill manually:

```sql
INSERT INTO events_daily_agg
SELECT
  toDate(event_time) AS event_date,
  event_type,
  countState() AS event_count,
  sumState(revenue) AS total_revenue,
  uniqState(user_id) AS unique_users
FROM events
GROUP BY event_date, event_type;
```

## Multi-Level Aggregation

Create a monthly rollup from the daily aggregate:

```sql
CREATE TABLE events_monthly_agg (
  event_month Date,
  event_type String,
  event_count AggregateFunction(count),
  total_revenue AggregateFunction(sum, Decimal64(2))
) ENGINE = AggregatingMergeTree()
ORDER BY (event_type, event_month);

-- Query combining both levels
SELECT
  toStartOfMonth(event_date) AS event_month,
  event_type,
  countMerge(event_count),
  sumMerge(total_revenue)
FROM events_daily_agg
WHERE event_date >= '2025-01-01'
GROUP BY event_month, event_type;
```

## Summary

Materialized views with `AggregatingMergeTree` pre-compute aggregations incrementally on every INSERT, making dashboard queries over billions of rows run in milliseconds. Use `-State` combinators when inserting and `-Merge` combinators when querying. Backfill existing data manually when adding views to populated tables.
