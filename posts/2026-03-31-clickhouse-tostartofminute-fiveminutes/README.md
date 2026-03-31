# How to Use toStartOfMinute() and toStartOfFiveMinutes() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Time Series, Analytics, Dashboard

Description: Learn how to use toStartOfMinute() and toStartOfFiveMinutes() in ClickHouse for high-resolution time series, candlestick charts, and minute-level aggregations.

---

When hourly bucketing is too coarse, ClickHouse offers `toStartOfMinute()` and `toStartOfFiveMinutes()` to snap timestamps to the nearest minute or 5-minute boundary. These functions are essential for real-time monitoring dashboards, financial candlestick charts, and any application where you need to visualize data at sub-hourly resolution.

This post explains both functions and demonstrates how to apply them to time-series queries, gap filling, and candlestick OHLC (Open/High/Low/Close) aggregations.

## toStartOfMinute() - Truncate to the Minute

`toStartOfMinute(dt)` returns a `DateTime` with the seconds component set to zero. The year, month, day, hour, and minute are preserved.

```sql
SELECT
    toDateTime('2026-03-31 14:22:47')                    AS input,
    toStartOfMinute(toDateTime('2026-03-31 14:22:47'))   AS minute_start;
```

```text
input                minute_start
-------------------- --------------------
2026-03-31 14:22:47  2026-03-31 14:22:00
```

## Per-Minute Request Rate

Group HTTP log entries by minute to produce a per-minute request rate suitable for a real-time chart:

```sql
SELECT
    toStartOfMinute(created_at)  AS minute,
    count()                      AS requests,
    countIf(status_code >= 500)  AS errors,
    avg(response_ms)             AS avg_response_ms
FROM http_logs
WHERE created_at >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;
```

## Minute-Level Gap Filling

Use `WITH FILL` to ensure every minute appears in the output, even during quiet periods:

```sql
SELECT
    toStartOfMinute(created_at)  AS minute,
    count()                      AS requests
FROM http_logs
WHERE created_at >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute ASC WITH FILL
    FROM toStartOfMinute(now() - INTERVAL 1 HOUR)
    TO   toStartOfMinute(now())
    STEP INTERVAL 1 MINUTE;
```

## Per-Minute Latency Percentiles

For SLO monitoring at minute resolution, compute latency percentiles per minute:

```sql
SELECT
    toStartOfMinute(created_at)       AS minute,
    count()                           AS requests,
    quantile(0.50)(response_ms)       AS p50_ms,
    quantile(0.95)(response_ms)       AS p95_ms,
    quantile(0.99)(response_ms)       AS p99_ms
FROM http_logs
WHERE created_at >= now() - INTERVAL 30 MINUTE
GROUP BY minute
ORDER BY minute;
```

## toStartOfFiveMinutes() - 5-Minute Buckets

`toStartOfFiveMinutes(dt)` snaps a timestamp to the start of the nearest 5-minute boundary. The boundaries are at :00, :05, :10, :15, and so on.

```sql
SELECT
    toDateTime('2026-03-31 14:22:47')                          AS input,
    toStartOfFiveMinutes(toDateTime('2026-03-31 14:22:47'))    AS five_min_start;
```

```text
input                five_min_start
-------------------- --------------------
2026-03-31 14:22:47  2026-03-31 14:20:00
```

A timestamp at 14:22:47 falls in the 14:20 bucket because 14:20 is the most recent :XX where XX is divisible by 5.

## 5-Minute Aggregation for a Monitoring Dashboard

5-minute intervals are a standard resolution for infrastructure monitoring dashboards:

```sql
SELECT
    toStartOfFiveMinutes(collected_at)  AS bucket,
    avg(cpu_usage_pct)                  AS avg_cpu,
    max(cpu_usage_pct)                  AS max_cpu,
    avg(memory_usage_pct)               AS avg_memory,
    avg(disk_io_mbps)                   AS avg_disk_io
FROM server_metrics
WHERE collected_at >= now() - INTERVAL 1 HOUR
GROUP BY bucket
ORDER BY bucket;
```

## Candlestick (OHLC) Chart with 5-Minute Buckets

Financial and price data is commonly displayed as OHLC candlestick charts. Each candle covers one 5-minute interval:

```sql
SELECT
    toStartOfFiveMinutes(trade_time)           AS candle_open_time,
    argMin(price, trade_time)                  AS open,
    max(price)                                 AS high,
    min(price)                                 AS low,
    argMax(price, trade_time)                  AS close,
    sum(volume)                                AS volume
FROM trades
WHERE symbol = 'BTC-USD'
  AND trade_time >= now() - INTERVAL 24 HOUR
GROUP BY candle_open_time
ORDER BY candle_open_time;
```

`argMin(price, trade_time)` returns the price of the first trade in the bucket (open), and `argMax(price, trade_time)` returns the price of the last trade (close).

## Comparing 1-Minute and 5-Minute Granularity

You can run both granularities in a single query using a UNION to let the application choose which resolution to display:

```sql
SELECT '1min' AS granularity, toStartOfMinute(created_at) AS bucket, count() AS requests
FROM http_logs
WHERE created_at >= now() - INTERVAL 30 MINUTE
GROUP BY bucket

UNION ALL

SELECT '5min' AS granularity, toStartOfFiveMinutes(created_at) AS bucket, count() AS requests
FROM http_logs
WHERE created_at >= now() - INTERVAL 3 HOUR
GROUP BY bucket

ORDER BY granularity, bucket;
```

## 5-Minute Gap Filling

Fill in missing 5-minute buckets so charts render without gaps:

```sql
SELECT
    toStartOfFiveMinutes(created_at)  AS bucket,
    count()                           AS requests
FROM http_logs
WHERE created_at >= now() - INTERVAL 3 HOUR
GROUP BY bucket
ORDER BY bucket ASC WITH FILL
    FROM toStartOfFiveMinutes(now() - INTERVAL 3 HOUR)
    TO   toStartOfFiveMinutes(now())
    STEP INTERVAL 5 MINUTE;
```

## Detecting Spikes at Minute Resolution

Identify 1-minute windows where the request rate was more than two standard deviations above the hourly mean:

```sql
WITH minute_counts AS (
    SELECT
        toStartOfMinute(created_at)  AS minute,
        count()                      AS requests
    FROM http_logs
    WHERE created_at >= now() - INTERVAL 1 HOUR
    GROUP BY minute
)
SELECT
    minute,
    requests,
    avg(requests) OVER ()              AS mean_requests,
    stddevPop(requests) OVER ()        AS stddev_requests,
    requests > avg(requests) OVER () + 2 * stddevPop(requests) OVER () AS is_spike
FROM minute_counts
ORDER BY minute;
```

## Summary

`toStartOfMinute()` and `toStartOfFiveMinutes()` are the right tools when hourly buckets are too coarse for your analysis. Use `toStartOfMinute()` for real-time dashboards, per-minute SLO monitoring, and anomaly detection at fine granularity. Use `toStartOfFiveMinutes()` for slightly smoother time series that reduce noise while still capturing short-lived spikes - it is the standard resolution for infrastructure dashboards and OHLC candlestick charts. Pair either function with `WITH FILL` to produce charts without gaps.
