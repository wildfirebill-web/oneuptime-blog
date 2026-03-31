# How to Implement Time Bucketing with Custom Intervals in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Time Bucketing, Time Series, Analytics, toStartOf

Description: Learn how to bucket time series data into custom intervals in ClickHouse using toStartOfInterval, intDiv, and arithmetic rounding techniques.

---

Grouping time-series data into fixed-size buckets is essential for charting, aggregating metrics, and building dashboards. ClickHouse provides several methods for custom interval bucketing.

## Using toStartOfInterval

`toStartOfInterval(ts, INTERVAL N unit)` is the most flexible approach:

```sql
SELECT
    toStartOfInterval(event_time, INTERVAL 15 MINUTE) AS bucket,
    count() AS requests,
    avg(response_ms) AS avg_latency
FROM http_requests
GROUP BY bucket
ORDER BY bucket;
```

Supported units include SECOND, MINUTE, HOUR, DAY, WEEK, MONTH, QUARTER, and YEAR.

## 5-Minute Buckets

```sql
SELECT
    toStartOfInterval(ts, INTERVAL 5 MINUTE) AS five_min_bucket,
    sum(bytes_transferred) AS total_bytes
FROM network_traffic
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY five_min_bucket
ORDER BY five_min_bucket;
```

## Custom Second-Level Buckets

For sub-minute buckets (e.g., every 30 seconds), use arithmetic:

```sql
SELECT
    toDateTime(intDiv(toUnixTimestamp(event_time), 30) * 30) AS bucket_30s,
    count() AS events
FROM sensor_data
GROUP BY bucket_30s
ORDER BY bucket_30s;
```

## Non-Standard Hour Intervals (e.g., 6-Hour Buckets)

```sql
SELECT
    toDateTime(intDiv(toUnixTimestamp(ts), 21600) * 21600) AS six_hour_bucket,
    avg(cpu_percent) AS avg_cpu
FROM host_metrics
GROUP BY six_hour_bucket
ORDER BY six_hour_bucket;
```

## Filling Gaps with zeros

Use a number series to fill empty buckets:

```sql
WITH
    toStartOfInterval(now() - INTERVAL 24 HOUR, INTERVAL 1 HOUR) AS start_time,
    now() AS end_time
SELECT
    bucket,
    ifNull(requests, 0) AS requests
FROM (
    SELECT arrayJoin(
        arrayMap(x -> toDateTime(toUnixTimestamp(start_time) + x * 3600), range(0, 24))
    ) AS bucket
) AS slots
LEFT JOIN (
    SELECT toStartOfInterval(event_time, INTERVAL 1 HOUR) AS bucket, count() AS requests
    FROM http_requests
    GROUP BY bucket
) AS data USING (bucket)
ORDER BY bucket;
```

## Summary

ClickHouse's `toStartOfInterval` handles most time bucketing needs. For sub-minute or unusual intervals, use `intDiv(toUnixTimestamp(ts), interval_seconds) * interval_seconds`. Fill gaps in sparse series using `arrayJoin` with a generated time range.
