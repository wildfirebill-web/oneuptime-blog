# How to Build Frequency Distribution Tables in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Frequency Distribution, Histogram, Analytics, SQL, Aggregation

Description: Learn how to build frequency distribution tables and histograms in ClickHouse using manual bucketing, histogram(), and width_bucket() functions.

---

Frequency distribution tables count how many values fall into each bucket or range. They are the foundation of histograms and are essential for understanding data distribution, latency percentiles, and user behavior patterns.

## Manual Bucket Approach

Use `multiIf` to assign values to named buckets:

```sql
SELECT
    multiIf(
        duration_ms < 50,    '< 50ms',
        duration_ms < 100,   '50-100ms',
        duration_ms < 200,   '100-200ms',
        duration_ms < 500,   '200-500ms',
        duration_ms < 1000,  '500ms-1s',
        duration_ms < 5000,  '1-5s',
        '> 5s'
    ) AS latency_bucket,
    count() AS requests,
    round(count() / sum(count()) OVER () * 100, 2) AS pct
FROM http_requests
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY latency_bucket
ORDER BY
    multiIf(
        latency_bucket = '< 50ms', 1,
        latency_bucket = '50-100ms', 2,
        latency_bucket = '100-200ms', 3,
        latency_bucket = '200-500ms', 4,
        latency_bucket = '500ms-1s', 5,
        latency_bucket = '1-5s', 6,
        7
    );
```

## Using histogram() Function

ClickHouse has a built-in `histogram()` aggregate function that automatically determines bucket boundaries:

```sql
SELECT histogram(10)(duration_ms) AS hist
FROM http_requests
WHERE timestamp >= now() - INTERVAL 1 HOUR;
```

This returns an array of `(lower, upper, height)` tuples. Expand into rows:

```sql
SELECT
    round(bucket.1, 0) AS lower_ms,
    round(bucket.2, 0) AS upper_ms,
    round(bucket.3) AS count
FROM (
    SELECT histogram(20)(duration_ms) AS hist
    FROM http_requests
    WHERE timestamp >= now() - INTERVAL 1 HOUR
)
ARRAY JOIN hist AS bucket
ORDER BY lower_ms;
```

## Using width_bucket() for Fixed-Width Buckets

```sql
-- Divide 0-1000ms range into 10 equal buckets
SELECT
    width_bucket(duration_ms, 0, 1000, 10) AS bucket_num,
    (bucket_num - 1) * 100 AS bucket_start_ms,
    bucket_num * 100 AS bucket_end_ms,
    count() AS requests
FROM http_requests
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY bucket_num
ORDER BY bucket_num;
```

## Log-Scale Bucketing for Wide Range Data

For latency data spanning milliseconds to minutes, use logarithmic buckets:

```sql
SELECT
    floor(log10(greatest(duration_ms, 1))) AS log_bucket,
    pow(10, floor(log10(greatest(duration_ms, 1)))) AS bucket_lower_ms,
    count() AS requests,
    round(count() / sum(count()) OVER () * 100, 2) AS pct
FROM http_requests
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY log_bucket
ORDER BY log_bucket;
```

## Cumulative Distribution

Add a cumulative column to see what percentage of requests are below each threshold:

```sql
SELECT
    latency_bucket,
    count,
    pct,
    sum(pct) OVER (ORDER BY bucket_order ROWS UNBOUNDED PRECEDING) AS cumulative_pct
FROM (
    SELECT
        multiIf(duration_ms < 100, '< 100ms', duration_ms < 500, '100-500ms',
                duration_ms < 1000, '500ms-1s', '> 1s') AS latency_bucket,
        multiIf(duration_ms < 100, 1, duration_ms < 500, 2,
                duration_ms < 1000, 3, 4) AS bucket_order,
        count() AS count,
        round(count() / sum(count()) OVER () * 100, 2) AS pct
    FROM http_requests
    GROUP BY latency_bucket, bucket_order
)
ORDER BY bucket_order;
```

## Summary

ClickHouse supports frequency distributions through manual bucketing with `multiIf`, automatic bucketing with `histogram()`, and equal-width buckets with `width_bucket()`. For wide-range data like latency, logarithmic bucketing provides better insight. Adding cumulative percentages transforms a frequency table into a CDF, showing what fraction of requests meet various latency thresholds.
