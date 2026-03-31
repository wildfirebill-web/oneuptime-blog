# How to Implement Time-Based Bucketing in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Time Bucketing, toStartOfInterval, Time-Series, Aggregation

Description: Learn how to implement flexible time-based bucketing in ClickHouse using toStartOfInterval, date functions, and custom bucket sizes for time-series analysis.

---

## Time Bucketing Fundamentals

Time bucketing groups events into fixed-size time intervals. ClickHouse provides a rich set of date/time functions for bucketing at any granularity, from seconds to months.

## Built-In Bucket Functions

```sql
-- By minute, hour, day, week, month
SELECT toStartOfMinute(ts) AS bucket FROM events;
SELECT toStartOfHour(ts) AS bucket FROM events;
SELECT toStartOfDay(ts) AS bucket FROM events;
SELECT toStartOfWeek(ts) AS bucket FROM events;
SELECT toStartOfMonth(ts) AS bucket FROM events;

-- 5-minute buckets
SELECT toStartOfFiveMinutes(ts) AS bucket FROM events;

-- 15-minute buckets
SELECT toStartOfFifteenMinutes(ts) AS bucket FROM events;
```

## Custom Bucket Sizes with toStartOfInterval

For arbitrary bucket sizes:

```sql
-- 10-minute buckets
SELECT toStartOfInterval(ts, INTERVAL 10 MINUTE) AS bucket,
    count() AS events
FROM events
WHERE ts >= now() - INTERVAL 24 HOUR
GROUP BY bucket
ORDER BY bucket;

-- 6-hour buckets
SELECT toStartOfInterval(ts, INTERVAL 6 HOUR) AS bucket,
    avg(latency_ms) AS avg_latency
FROM requests
GROUP BY bucket
ORDER BY bucket;
```

## Dynamic Bucket Size

Adjust bucket size based on the queried range to keep response times consistent:

```sql
-- For a 24-hour window, use 5-minute buckets
SELECT
    toStartOfInterval(event_time, INTERVAL 5 MINUTE) AS bucket,
    count() AS events
FROM events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY bucket
ORDER BY bucket;

-- For a 7-day window, use hourly buckets
SELECT
    toStartOfHour(event_time) AS bucket,
    count() AS events
FROM events
WHERE event_time >= now() - INTERVAL 7 DAY
GROUP BY bucket
ORDER BY bucket;
```

## Unix Timestamp Bucketing

For sub-second precision with millisecond timestamps:

```sql
-- 100ms buckets
SELECT
    toDateTime64(
        intDiv(toUnixTimestamp64Milli(ts), 100) * 100 / 1000, 3
    ) AS bucket,
    count() AS events
FROM high_frequency_events
GROUP BY bucket
ORDER BY bucket;
```

## Business Calendar Bucketing

Bucket by fiscal quarter:

```sql
SELECT
    concat(toString(toYear(event_time)), '-Q',
        toString(toQuarter(event_time))) AS fiscal_quarter,
    sum(revenue) AS quarterly_revenue
FROM sales
WHERE event_time >= toDate('2025-01-01')
GROUP BY fiscal_quarter
ORDER BY fiscal_quarter;
```

## Heatmap Bucketing

Create a day-of-week vs. hour-of-day heatmap:

```sql
SELECT
    toDayOfWeek(event_time) AS day_of_week,
    toHour(event_time) AS hour,
    count() AS events
FROM events
WHERE event_time >= today() - 30
GROUP BY day_of_week, hour
ORDER BY day_of_week, hour;
```

## Summary

ClickHouse's time bucketing functions - from `toStartOfMinute` to `toStartOfInterval` - cover all common granularities. Custom millisecond bucketing handles high-frequency data, and combining day-of-week with hour-of-day buckets enables heatmap analysis. Dynamic bucket sizing based on query range keeps dashboards responsive at any scale.
