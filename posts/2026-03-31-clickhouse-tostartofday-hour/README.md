# How to Use toStartOfDay() and toStartOfHour() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Time Series, Analytics, Dashboard

Description: Learn how to use toStartOfDay() and toStartOfHour() in ClickHouse for daily and hourly time-series bucketing in analytics and dashboard queries.

---

Time-series analytics almost always requires bucketing raw timestamps into uniform intervals. ClickHouse provides `toStartOfDay()` and `toStartOfHour()` to snap a `DateTime` value to the beginning of its containing day or hour. The result is a clean, sortable label that works directly in `GROUP BY` and generates consistently-spaced time series for charts and dashboards.

This post explains both functions in detail and walks through the most important query patterns: daily aggregations, hourly aggregations, filling gaps, and per-interval error rate calculations.

## toStartOfDay() - Truncate to Midnight

`toStartOfDay(dt)` returns a `DateTime` value with the time component set to `00:00:00`. In other words, it returns the midnight that begins the calendar day of the input.

```sql
SELECT
    toDateTime('2026-03-31 14:22:05')                AS input,
    toStartOfDay(toDateTime('2026-03-31 14:22:05'))  AS day_start;
```

```text
input                day_start
-------------------- --------------------
2026-03-31 14:22:05  2026-03-31 00:00:00
```

Note that `toStartOfDay()` accepts a `DateTime` and returns a `DateTime` (at midnight), not a `Date`. If you need a `Date` value, wrap it with `toDate()` or use `today()` / `toDate(dt)` directly.

## Daily Aggregations

The most common use of `toStartOfDay()` is grouping events by calendar day for time-series charts:

```sql
SELECT
    toStartOfDay(created_at)  AS day,
    count()                   AS requests,
    countIf(status_code >= 500) AS server_errors,
    avg(response_ms)          AS avg_response_ms
FROM http_logs
WHERE created_at >= now() - INTERVAL 30 DAY
GROUP BY day
ORDER BY day;
```

## Daily Unique Users

Count distinct users per day:

```sql
SELECT
    toStartOfDay(event_time)  AS day,
    uniq(user_id)             AS daily_active_users
FROM user_events
WHERE event_time >= now() - INTERVAL 90 DAY
GROUP BY day
ORDER BY day;
```

## toStartOfHour() - Truncate to the Hour

`toStartOfHour(dt)` returns a `DateTime` with the minutes and seconds set to zero, keeping the year, month, day, and hour intact.

```sql
SELECT
    toDateTime('2026-03-31 14:22:05')                 AS input,
    toStartOfHour(toDateTime('2026-03-31 14:22:05'))  AS hour_start;
```

```text
input                hour_start
-------------------- --------------------
2026-03-31 14:22:05  2026-03-31 14:00:00
```

## Hourly Aggregations

Hourly bucketing is the workhorse of infrastructure monitoring dashboards. Group request counts and latency by hour:

```sql
SELECT
    toStartOfHour(created_at)  AS hour,
    count()                    AS requests,
    quantile(0.99)(response_ms) AS p99_response_ms,
    countIf(status_code >= 500) AS server_errors
FROM http_logs
WHERE created_at >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;
```

## Combining Daily and Hourly Views

A single query can expose both granularities using two separate subqueries or a parameterized approach. For a dashboard that shows hourly data for today and daily data for the past 30 days, run them as separate queries and union the results:

```sql
-- Hourly for today
SELECT
    toStartOfHour(created_at)  AS bucket,
    'hourly'                   AS granularity,
    count()                    AS requests
FROM http_logs
WHERE created_at >= toStartOfDay(now())
GROUP BY bucket

UNION ALL

-- Daily for the past 30 days (excluding today)
SELECT
    toStartOfDay(created_at)   AS bucket,
    'daily'                    AS granularity,
    count()                    AS requests
FROM http_logs
WHERE created_at >= now() - INTERVAL 30 DAY
  AND created_at < toStartOfDay(now())
GROUP BY bucket

ORDER BY bucket;
```

## Filling Gaps in Time Series

ClickHouse's `WITH FILL` modifier ensures that every interval appears in the output, even if there are no matching rows. This prevents gaps in charts:

```sql
SELECT
    toStartOfHour(created_at)  AS hour,
    count()                    AS requests
FROM http_logs
WHERE created_at >= now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour ASC WITH FILL
    FROM toStartOfHour(now() - INTERVAL 24 HOUR)
    TO   toStartOfHour(now())
    STEP INTERVAL 1 HOUR;
```

The same pattern works for daily fills:

```sql
SELECT
    toStartOfDay(created_at)  AS day,
    count()                   AS events
FROM user_events
WHERE created_at >= now() - INTERVAL 30 DAY
GROUP BY day
ORDER BY day ASC WITH FILL
    FROM toStartOfDay(now() - INTERVAL 30 DAY)
    TO   toStartOfDay(now())
    STEP INTERVAL 1 DAY;
```

## Hourly Error Rate for SLO Monitoring

Compute the error rate for each hour to feed an SLO compliance dashboard:

```sql
SELECT
    toStartOfHour(created_at)                                         AS hour,
    count()                                                            AS total_requests,
    countIf(status_code >= 500)                                        AS error_count,
    round(countIf(status_code >= 500) / count() * 100, 4)             AS error_rate_pct,
    if(countIf(status_code >= 500) / count() < 0.001, 'OK', 'BREACH') AS slo_status
FROM http_logs
WHERE created_at >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;
```

## Daily Revenue with Day-over-Day Delta

Compare each day's revenue to the previous day using `lagInFrame`:

```sql
SELECT
    day,
    daily_revenue,
    lagInFrame(daily_revenue) OVER (ORDER BY day) AS prev_day_revenue,
    daily_revenue - lagInFrame(daily_revenue) OVER (ORDER BY day) AS delta
FROM (
    SELECT
        toStartOfDay(sale_date)  AS day,
        sum(revenue)             AS daily_revenue
    FROM sales
    WHERE sale_date >= now() - INTERVAL 30 DAY
    GROUP BY day
)
ORDER BY day;
```

## Summary

`toStartOfDay()` and `toStartOfHour()` are the two most frequently used time-bucketing functions for time-series analytics in ClickHouse. Use `toStartOfDay()` for daily aggregations, day-over-day comparisons, and daily active user counts. Use `toStartOfHour()` for infrastructure monitoring, hourly SLO reporting, and per-hour latency distributions. Pair either function with `WITH FILL` to produce gap-free time series ready for charting.
