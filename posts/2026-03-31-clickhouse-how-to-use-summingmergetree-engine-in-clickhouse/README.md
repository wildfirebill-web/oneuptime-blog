# How to Use SummingMergeTree Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SummingMergeTree, Table Engine, Aggregation, SQL

Description: Learn how to use SummingMergeTree in ClickHouse to automatically sum numeric columns during background merges for efficient pre-aggregated storage.

---

## Overview

`SummingMergeTree` is a ClickHouse engine that automatically sums numeric columns for rows sharing the same primary key during background merges. Instead of storing individual events, it stores running totals that ClickHouse collapses over time. It is ideal for pre-aggregated metrics like daily counters, revenue totals, and request counts.

## Creating a SummingMergeTree Table

```sql
CREATE TABLE daily_metrics
(
    date        Date,
    service     LowCardinality(String),
    region      LowCardinality(String),
    requests    UInt64,
    errors      UInt64,
    latency_sum Float64
)
ENGINE = SummingMergeTree()
ORDER BY (date, service, region);
```

All numeric columns not in the `ORDER BY` are summed. The `ORDER BY` key defines the grouping dimensions.

## Inserting Data

You can insert partial data - incremental counters from different time windows or batches - and SummingMergeTree will consolidate them:

```sql
INSERT INTO daily_metrics VALUES
('2024-06-01', 'api', 'us-east', 1000, 5, 2500.0),
('2024-06-01', 'api', 'us-east',  500, 2, 1200.0),  -- same key, will be summed
('2024-06-01', 'api', 'eu-west', 2000, 8, 4100.0);
```

## Reading with SUM + GROUP BY

Until merges have consolidated rows, always use `SUM` with `GROUP BY` to get correct totals:

```sql
SELECT
    date,
    service,
    region,
    sum(requests)    AS total_requests,
    sum(errors)      AS total_errors,
    sum(latency_sum) AS total_latency
FROM daily_metrics
WHERE date = '2024-06-01'
GROUP BY date, service, region
ORDER BY date, service, region
```

This pattern works correctly before and after merges.

## Specifying Which Columns to Sum

Pass a tuple to `SummingMergeTree` to control which columns are summed - others retain their last-written value:

```sql
CREATE TABLE page_views
(
    date        Date,
    page        String,
    views       UInt64,
    unique_ips  UInt64,
    page_title  String  -- NOT summed; will retain last value
)
ENGINE = SummingMergeTree((views, unique_ips))
ORDER BY (date, page);
```

## Partitioning for Efficient Merges

Add partition keys to limit merge scope and improve performance:

```sql
CREATE TABLE event_counts
(
    event_date   Date,
    event_type   LowCardinality(String),
    count        UInt64
)
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_type);
```

## Combining with Materialized Views

A common pattern is to use `SummingMergeTree` as the target of a materialized view that pre-aggregates a raw events table:

```sql
CREATE MATERIALIZED VIEW mv_daily_events
TO daily_metrics
AS
SELECT
    toDate(event_time) AS date,
    service,
    region,
    count()            AS requests,
    countIf(status >= 500) AS errors,
    sum(response_ms)   AS latency_sum
FROM raw_events
GROUP BY date, service, region;
```

## Forcing Merge for Testing

```sql
OPTIMIZE TABLE daily_metrics FINAL;
```

After this, duplicate keys will have been summed into single rows.

## Limitations

- Summing only happens at merge time - duplicate rows may coexist temporarily.
- Non-numeric, non-key columns retain an arbitrary value (not a sum). Be explicit about which columns to sum.
- Not suitable if you need exact per-event data - use a raw MergeTree for that.

## Summary

`SummingMergeTree` automatically sums numeric columns sharing the same sorting key during background merges, making it ideal for pre-aggregated metric storage. Always read with `SUM + GROUP BY` to handle partially merged data, use materialized views to populate it from raw events, and specify which columns to sum explicitly when non-metric columns are present.
