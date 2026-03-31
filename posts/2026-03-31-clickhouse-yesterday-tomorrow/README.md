# How to Use yesterday() and tomorrow() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Analytics, Reporting, Query Filter

Description: Learn how yesterday() and tomorrow() return the previous and next calendar days as Date values, simplifying daily reporting filters and scheduling window queries.

---

ClickHouse provides two simple zero-argument date functions: `yesterday()` returns the `Date` value for the day before the current server date, and `tomorrow()` returns the `Date` value for the day after. Both return a `Date` type (not a `DateTime`), so they represent the full day without a time component. These functions are shorthand for `today() - 1` and `today() + 1` respectively, and they make daily reporting queries more readable by making the intent immediately obvious.

## Basic Usage

```sql
-- See the return values relative to today
SELECT
    yesterday() AS yesterday_date,
    today()     AS today_date,
    tomorrow()  AS tomorrow_date;
```

```text
yesterday_date  today_date  tomorrow_date
2024-06-14      2024-06-15  2024-06-16
```

Both functions respect the server's local date. If your server runs in UTC, "today" is the UTC calendar day.

## Simple Yesterday Filter

The most common use is a filter for rows created or occurring yesterday.

```sql
-- Count yesterday's orders by status
SELECT
    status,
    count()      AS order_count,
    sum(amount)  AS total_revenue
FROM orders
WHERE toDate(created_at) = yesterday()
GROUP BY status
ORDER BY order_count DESC;
```

## Yesterday vs Today Comparison

Comparing yesterday's metrics to today's is a natural use of both functions together.

```sql
-- Side-by-side comparison of today vs yesterday
SELECT
    'yesterday' AS period,
    count()     AS events,
    uniq(user_id) AS unique_users
FROM user_events
WHERE event_date = yesterday()

UNION ALL

SELECT
    'today' AS period,
    count(),
    uniq(user_id)
FROM user_events
WHERE event_date = today();
```

## Tomorrow as a Scheduling Window Boundary

`tomorrow()` is useful when querying tasks or events scheduled to start before the end of today.

```sql
-- Find all scheduled jobs that run before tomorrow (i.e., today or earlier)
SELECT
    job_id,
    job_name,
    scheduled_at,
    status
FROM scheduled_jobs
WHERE
    toDate(scheduled_at) < tomorrow()
    AND status = 'pending'
ORDER BY scheduled_at;
```

## Daily Report: Yesterday's Top Pages

```sql
-- Top 10 most-visited pages yesterday
SELECT
    page_url,
    count()           AS page_views,
    uniq(visitor_id)  AS unique_visitors,
    avg(load_time_ms) AS avg_load_time_ms
FROM page_views
WHERE event_date = yesterday()
GROUP BY page_url
ORDER BY page_views DESC
LIMIT 10;
```

## Using yesterday() With DateTime Columns

Because `yesterday()` returns a `Date`, comparing it directly with a `DateTime` column uses implicit casting. For clarity, explicitly construct the range.

```sql
-- Explicit DateTime range for yesterday using yesterday() boundaries
SELECT
    event_id,
    event_type,
    event_time,
    user_id
FROM events
WHERE
    event_time >= toDateTime(yesterday())               -- start of yesterday
    AND event_time < toDateTime(today())                -- start of today
ORDER BY event_time;
```

This approach is more reliable than `toDate(event_time) = yesterday()` when the table uses a `DateTime` sort key, because the explicit range can use the primary index directly.

## Generating a Rolling 3-Day Window

Combine `yesterday()`, `today()`, and `tomorrow()` to build a simple rolling window around the current date.

```sql
-- Events from yesterday, today, and tomorrow (e.g., for a scheduling calendar)
SELECT
    toDate(event_time) AS event_day,
    event_type,
    count() AS event_count
FROM events
WHERE
    toDate(event_time) BETWEEN yesterday() AND tomorrow()
GROUP BY event_day, event_type
ORDER BY event_day, event_count DESC;
```

## Checking Yesterday's Data Completeness

A common data pipeline health check verifies that a minimum number of rows arrived for yesterday's date.

```sql
-- Alert if yesterday's row count is below expected threshold
SELECT
    yesterday() AS check_date,
    count()     AS row_count,
    count() < 100000 AS below_threshold
FROM events
WHERE event_date = yesterday();
```

## Equivalent Expressions

`yesterday()` and `tomorrow()` are syntactic sugar over arithmetic on `today()`.

```sql
-- All three expressions return the same Date
SELECT
    yesterday()    AS using_function,
    today() - 1    AS using_subtraction,
    today() - toIntervalDay(1) AS using_interval;
```

The function form is preferred for readability in production queries.

## Summary

`yesterday()` and `tomorrow()` are convenience functions that return `Date` values for the day before and after the current date. They make daily report filters and scheduling window queries more readable than manual arithmetic. When filtering `DateTime` columns, prefer explicit range predicates (`>= toDateTime(yesterday()) AND < toDateTime(today())`) over `toDate(dt) = yesterday()` to allow ClickHouse to use primary index range scans efficiently.
