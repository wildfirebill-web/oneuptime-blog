# How to Use toStartOfInterval() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Time Series, Aggregation, Interval

Description: Learn how to use toStartOfInterval() to bucket DateTime values into arbitrary intervals like 5 minutes or 6 hours for flexible time series aggregation.

---

`toStartOfInterval()` is one of the most useful date functions in ClickHouse for time series analysis. It truncates a `DateTime` or `DateTime64` value to the start of a user-defined interval, letting you group events into any bucket size without writing custom arithmetic. This makes it far more flexible than the fixed-interval helpers like `toStartOfFiveMinutes()` or `toStartOfHour()`.

## Function Signature

The function takes two required arguments: a DateTime value and an INTERVAL expression.

```text
toStartOfInterval(dt, INTERVAL n unit)
toStartOfInterval(dt, INTERVAL n unit, timezone)
```

Supported units include: `SECOND`, `MINUTE`, `HOUR`, `DAY`, `WEEK`, `MONTH`, `QUARTER`, and `YEAR`. The optional third argument lets you specify a timezone so interval boundaries align to local midnight or other locale-specific boundaries.

## Basic Usage

The simplest use case is snapping a timestamp to the start of the enclosing interval. The query below truncates a timestamp to the nearest 5-minute boundary.

```sql
SELECT
    toStartOfInterval(toDateTime('2026-03-31 14:23:47'), INTERVAL 5 MINUTE) AS bucket;
-- Result: 2026-03-31 14:20:00
```

You can use any numeric multiplier for the interval. For example, to bucket into 15-minute windows:

```sql
SELECT
    toStartOfInterval(toDateTime('2026-03-31 14:23:47'), INTERVAL 15 MINUTE) AS bucket;
-- Result: 2026-03-31 14:15:00
```

## Aggregating Events into 5-Minute Buckets

A common use case is counting events per time bucket to build a time series chart. The query below groups HTTP requests into 5-minute windows and counts each bucket.

```sql
SELECT
    toStartOfInterval(event_time, INTERVAL 5 MINUTE) AS bucket,
    count()                                           AS request_count
FROM http_logs
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY bucket
ORDER BY bucket ASC;
```

This is equivalent to writing `floor(toUnixTimestamp(event_time) / 300) * 300` but much more readable and timezone-aware.

## Using Larger Intervals: 6-Hour Buckets

You are not limited to minutes. The query below divides the day into four 6-hour windows, which is useful for shift-based reporting.

```sql
SELECT
    toStartOfInterval(event_time, INTERVAL 6 HOUR) AS shift_start,
    sum(revenue)                                    AS shift_revenue,
    count()                                         AS order_count
FROM orders
WHERE event_time >= toStartOfDay(today())
GROUP BY shift_start
ORDER BY shift_start ASC;
```

## Timezone-Aware Bucketing

When your users are spread across time zones, bucket boundaries should align to the user's local time rather than UTC. Pass the timezone as the third argument.

```sql
SELECT
    toStartOfInterval(event_time, INTERVAL 1 DAY, 'America/New_York') AS day_bucket,
    count()                                                            AS events
FROM user_activity
WHERE user_timezone = 'America/New_York'
  AND event_time >= now() - INTERVAL 7 DAY
GROUP BY day_bucket
ORDER BY day_bucket ASC;
```

Without the timezone argument, the day boundary falls at midnight UTC, which may split a user's activity day into two buckets.

## Dynamic Interval Bucketing with Parameters

ClickHouse query parameters let you pass the interval size at query time, making dashboards configurable without changing the SQL.

```sql
-- Set the interval multiplier as a query parameter:
-- clickhouse-client --param_interval_minutes=10 ...

SELECT
    toStartOfInterval(event_time, INTERVAL {interval_minutes:UInt32} MINUTE) AS bucket,
    avg(response_ms) AS avg_latency
FROM api_requests
WHERE event_time >= now() - INTERVAL 2 HOUR
GROUP BY bucket
ORDER BY bucket ASC;
```

## Comparing toStartOfInterval vs Fixed Helpers

ClickHouse ships with convenience functions for common intervals, but they cannot be parameterised. The table below summarises the trade-offs.

```text
Fixed helper                   Equivalent toStartOfInterval call
toStartOfMinute(dt)            toStartOfInterval(dt, INTERVAL 1 MINUTE)
toStartOfFiveMinutes(dt)       toStartOfInterval(dt, INTERVAL 5 MINUTE)
toStartOfTenMinutes(dt)        toStartOfInterval(dt, INTERVAL 10 MINUTE)
toStartOfFifteenMinutes(dt)    toStartOfInterval(dt, INTERVAL 15 MINUTE)
toStartOfHour(dt)              toStartOfInterval(dt, INTERVAL 1 HOUR)
toStartOfDay(dt)               toStartOfInterval(dt, INTERVAL 1 DAY)
```

Use the fixed helpers when the interval is constant and you want maximum clarity. Use `toStartOfInterval()` when you need arbitrary bucket sizes, runtime parameterisation, or timezone alignment.

## Building a Latency Heatmap

A latency heatmap requires bucketing both time and latency. `toStartOfInterval()` handles the time axis while `intDiv` handles the latency axis.

```sql
SELECT
    toStartOfInterval(event_time, INTERVAL 10 MINUTE) AS time_bucket,
    intDiv(response_ms, 50) * 50                       AS latency_bucket,
    count()                                            AS requests
FROM api_requests
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY time_bucket, latency_bucket
ORDER BY time_bucket ASC, latency_bucket ASC;
```

## Summary

`toStartOfInterval()` is the most flexible time bucketing function in ClickHouse. It accepts any numeric interval, supports all major time units, and respects timezones - making it suitable for everything from real-time dashboards at 1-minute granularity to monthly billing summaries. Prefer it over fixed helpers whenever the interval might change or needs to be passed as a parameter.
