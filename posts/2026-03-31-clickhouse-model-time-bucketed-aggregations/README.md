# How to Model Time-Bucketed Aggregations in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Time Bucket, Aggregation, toStartOfInterval, Time Series, Analytics

Description: Learn how to model time-bucketed aggregations in ClickHouse using toStartOfInterval and related functions for flexible time-window analytics.

---

Time-bucketed aggregations group events into fixed time windows - the foundation of any time-series dashboard. ClickHouse provides a rich set of time bucketing functions that work seamlessly with its columnar storage.

## Core Time Bucketing Functions

```sql
-- Fixed intervals
SELECT toStartOfMinute(timestamp)      -- minute boundary
SELECT toStartOfFiveMinutes(timestamp) -- 5-minute boundary
SELECT toStartOfTenMinutes(timestamp)  -- 10-minute boundary
SELECT toStartOfHour(timestamp)        -- hour boundary
SELECT toStartOfDay(timestamp)         -- day boundary
SELECT toStartOfWeek(timestamp)        -- week boundary
SELECT toStartOfMonth(timestamp)       -- month boundary
SELECT toStartOfYear(timestamp)        -- year boundary

-- Dynamic interval
SELECT toStartOfInterval(timestamp, INTERVAL 15 MINUTE)
SELECT toStartOfInterval(timestamp, INTERVAL 4 HOUR)
```

## Basic Time-Bucketed Aggregation

```sql
SELECT
    toStartOfInterval(timestamp, INTERVAL {interval:UInt32} SECOND) AS bucket,
    service,
    count() AS requests,
    avg(duration_ms) AS avg_latency,
    quantile(0.95)(duration_ms) AS p95_latency,
    countIf(status_code >= 500) AS errors
FROM http_requests
WHERE timestamp >= now() - INTERVAL 24 HOUR
  AND service = 'api-gateway'
GROUP BY bucket, service
ORDER BY bucket;
```

## Filling Empty Buckets

Time-series queries often need zero values for buckets with no data:

```sql
WITH
    buckets AS (
        SELECT toStartOfMinute(now() - toIntervalMinute(number)) AS bucket
        FROM numbers(60)
    ),
    actual AS (
        SELECT
            toStartOfMinute(timestamp) AS bucket,
            count() AS requests
        FROM http_requests
        WHERE timestamp >= now() - INTERVAL 1 HOUR
        GROUP BY bucket
    )
SELECT
    b.bucket,
    coalesce(a.requests, 0) AS requests
FROM buckets b
LEFT JOIN actual a ON b.bucket = a.bucket
ORDER BY b.bucket;
```

## Sliding Window Aggregations

Use window functions for moving averages:

```sql
SELECT
    bucket,
    requests,
    avg(requests) OVER (
        ORDER BY bucket
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) AS requests_5min_moving_avg
FROM (
    SELECT
        toStartOfMinute(timestamp) AS bucket,
        count() AS requests
    FROM http_requests
    WHERE timestamp >= now() - INTERVAL 1 HOUR
    GROUP BY bucket
)
ORDER BY bucket;
```

## Multi-Resolution Bucketing

Serve different time ranges from different granularities:

```sql
-- Auto-select granularity based on range
SELECT
    multiIf(
        dateDiff('hour', start_time, end_time) <= 1,
        toStartOfMinute(timestamp),
        dateDiff('hour', start_time, end_time) <= 24,
        toStartOfHour(timestamp),
        toStartOfDay(timestamp)
    ) AS bucket,
    count() AS events
FROM events
CROSS JOIN (SELECT now() - INTERVAL 6 HOUR AS start_time, now() AS end_time) params
WHERE timestamp BETWEEN start_time AND end_time
GROUP BY bucket
ORDER BY bucket;
```

## Percentile Over Time Buckets

```sql
SELECT
    toStartOfHour(timestamp) AS hour,
    service,
    quantiles(0.50, 0.90, 0.95, 0.99)(duration_ms) AS percentiles
FROM http_requests
WHERE timestamp >= now() - INTERVAL 7 DAY
GROUP BY hour, service
ORDER BY hour, service;
```

Access array elements by index:

```sql
SELECT
    hour,
    percentiles[1] AS p50,
    percentiles[2] AS p90,
    percentiles[3] AS p95,
    percentiles[4] AS p99
FROM (
    SELECT
        toStartOfHour(timestamp) AS hour,
        quantiles(0.50, 0.90, 0.95, 0.99)(duration_ms) AS percentiles
    FROM http_requests
    GROUP BY hour
)
ORDER BY hour;
```

## Summary

ClickHouse's `toStartOfInterval` and related time functions are the building blocks for time-bucketed aggregations. Combine them with `numbers()` for gap-filling, window functions for moving averages, and parameterized views for reusable multi-resolution dashboards. These patterns cover nearly every time-series visualization requirement.
