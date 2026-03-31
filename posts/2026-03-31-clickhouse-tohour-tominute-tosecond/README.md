# How to Use toHour(), toMinute(), toSecond() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Time Function, Analytics, Aggregation

Description: Learn how to extract hour, minute, and second components from DateTime values in ClickHouse for time-of-day filtering and hourly aggregations.

---

When you need to analyze data by time of day rather than by calendar date, ClickHouse provides `toHour()`, `toMinute()`, and `toSecond()` to extract the time components from a `DateTime` or `DateTime64` value. These functions return integers you can use directly in `WHERE` clauses, `GROUP BY` expressions, and computed columns.

This post walks through each function, explains its return range, and shows real-world query patterns including peak-hour analysis, time-of-day filtering, and high-resolution aggregations.

## toHour() - Extract the Hour

`toHour(dt)` returns a `UInt8` in the range 0 to 23 representing the hour of the day in the server's local timezone.

```sql
SELECT toHour(toDateTime('2026-03-31 14:22:05')) AS hour;
```

```text
hour
-----
14
```

Use `toHour()` to bucket events by hour of the day, collapsing across all dates. This reveals intra-day patterns:

```sql
SELECT
    toHour(created_at)  AS hour_of_day,
    count()             AS requests,
    avg(response_ms)    AS avg_response_ms
FROM http_logs
WHERE created_at >= now() - INTERVAL 7 DAY
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

Filter for business hours only (9 AM to 5 PM):

```sql
SELECT
    created_at,
    user_id,
    action
FROM user_events
WHERE toHour(created_at) BETWEEN 9 AND 17
ORDER BY created_at DESC;
```

Identify the top 3 peak hours by request volume:

```sql
SELECT
    toHour(created_at)  AS hour_of_day,
    count()             AS requests
FROM http_logs
GROUP BY hour_of_day
ORDER BY requests DESC
LIMIT 3;
```

## toMinute() - Extract the Minute

`toMinute(dt)` returns a `UInt8` in the range 0 to 59 representing the minute within the hour.

```sql
SELECT toMinute(toDateTime('2026-03-31 14:22:05')) AS minute;
```

```text
minute
-------
22
```

Use `toMinute()` to find which minutes within any given hour have the most activity - useful for detecting cron jobs or scheduled batch writes:

```sql
SELECT
    toMinute(created_at)  AS minute_of_hour,
    count()               AS events
FROM task_executions
WHERE toYear(created_at) = 2026
GROUP BY minute_of_hour
ORDER BY minute_of_hour;
```

Combine `toHour()` and `toMinute()` to bucket events into specific time slots:

```sql
SELECT
    toHour(event_time)   AS hour,
    toMinute(event_time) AS minute,
    count()              AS occurrences
FROM sensor_readings
WHERE event_time >= today()
GROUP BY hour, minute
ORDER BY hour, minute;
```

## toSecond() - Extract the Second

`toSecond(dt)` returns a `UInt8` in the range 0 to 59 representing the second within the minute.

```sql
SELECT toSecond(toDateTime('2026-03-31 14:22:05')) AS second;
```

```text
second
-------
5
```

A practical use for `toSecond()` is detecting whether events cluster at the start of a minute (second = 0), which is typical of scheduled jobs:

```sql
SELECT
    toSecond(created_at)  AS second_of_minute,
    count()               AS events
FROM cron_events
GROUP BY second_of_minute
ORDER BY events DESC
LIMIT 10;
```

For `DateTime64` columns with sub-second precision, note that `toSecond()` still returns an integer; use `toMillisecond()` or arithmetic on the `DateTime64` value for sub-second components.

## Hourly Aggregation with Hour-of-Day Labels

A common dashboard query aggregates metrics by hour and formats the label for display:

```sql
SELECT
    concat(leftPad(toString(toHour(created_at)), 2, '0'), ':00') AS hour_label,
    count()                                                        AS requests,
    countIf(status_code >= 500)                                    AS server_errors,
    avg(response_ms)                                               AS avg_response_ms
FROM http_logs
WHERE created_at >= now() - INTERVAL 24 HOUR
GROUP BY hour_label
ORDER BY hour_label;
```

## Peak vs. Off-Peak Traffic Classification

Classify each request as peak (8 AM - 10 PM) or off-peak and compare error rates:

```sql
SELECT
    if(toHour(created_at) BETWEEN 8 AND 22, 'Peak', 'Off-Peak') AS period,
    count()                                                        AS requests,
    countIf(status_code >= 500)                                    AS errors,
    round(countIf(status_code >= 500) / count() * 100, 2)         AS error_rate_pct
FROM http_logs
WHERE created_at >= now() - INTERVAL 30 DAY
GROUP BY period
ORDER BY period;
```

## Time-of-Day Heatmap Data

Generate data suitable for a day-of-week by hour-of-day heatmap:

```sql
SELECT
    toDayOfWeek(created_at)  AS day_of_week,
    toHour(created_at)       AS hour_of_day,
    count()                  AS events
FROM user_events
WHERE created_at >= now() - INTERVAL 90 DAY
GROUP BY day_of_week, hour_of_day
ORDER BY day_of_week, hour_of_day;
```

## Summary

`toHour()`, `toMinute()`, and `toSecond()` let you dissect the time portion of any `DateTime` value into queryable integers. Use `toHour()` for coarse time-of-day analysis and peak detection, `toMinute()` for finding intra-hour patterns like scheduled tasks, and `toSecond()` for detecting burst patterns at second resolution. Combining these functions with standard aggregations makes it straightforward to build hourly dashboards, time-of-day heatmaps, and traffic classification queries.
