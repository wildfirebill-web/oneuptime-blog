# How to Use HOUR(), MINUTE(), SECOND() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Hour Minute Second, Date Function, Time Function, SQL

Description: Learn how to use HOUR(), MINUTE(), and SECOND() functions in MySQL to extract time components from TIME, DATETIME, and TIMESTAMP values.

---

## Introduction

MySQL provides `HOUR()`, `MINUTE()`, and `SECOND()` functions to extract individual time components from TIME, DATETIME, and TIMESTAMP values. These are essential for time-based filtering, grouping by time period, scheduling logic, and calculating durations.

## HOUR() Function

Returns the hour component (0-23 for TIME/DATETIME, or larger for TIME intervals):

```sql
SELECT HOUR('14:30:45');               -- Returns: 14
SELECT HOUR('2024-01-15 09:25:30');    -- Returns: 9
SELECT HOUR(NOW());                    -- Returns: current hour
SELECT HOUR('100:30:00');              -- Returns: 100 (TIME supports > 24h)
```

## MINUTE() Function

Returns the minute component (0-59):

```sql
SELECT MINUTE('14:30:45');             -- Returns: 30
SELECT MINUTE('2024-01-15 09:25:30'); -- Returns: 25
SELECT MINUTE(NOW());                  -- Returns: current minute
```

## SECOND() Function

Returns the seconds component (0-59):

```sql
SELECT SECOND('14:30:45');             -- Returns: 45
SELECT SECOND('2024-01-15 09:25:30'); -- Returns: 30
SELECT SECOND(NOW());                  -- Returns: current second
```

## Using All Three Together

```sql
SELECT
  created_at,
  HOUR(created_at)   AS hour,
  MINUTE(created_at) AS minute,
  SECOND(created_at) AS second
FROM events;
```

## Filtering by Hour

Find all orders placed during business hours (9 AM to 5 PM):

```sql
SELECT id, customer_id, total_amount, created_at
FROM orders
WHERE HOUR(created_at) BETWEEN 9 AND 17;
```

Find late-night activity (midnight to 4 AM):

```sql
SELECT user_id, action, timestamp
FROM activity_log
WHERE HOUR(timestamp) BETWEEN 0 AND 4;
```

## Filtering by Minute

Find events that occurred at the top of any hour:

```sql
SELECT id, name, scheduled_at
FROM events
WHERE MINUTE(scheduled_at) = 0;
```

## Grouping by Hour

Count events per hour of day:

```sql
SELECT
  HOUR(created_at) AS hour_of_day,
  COUNT(*) AS event_count
FROM events
WHERE DATE(created_at) = CURDATE()
GROUP BY HOUR(created_at)
ORDER BY hour_of_day;
```

## Grouping by Hour and Minute (for 15-minute buckets)

```sql
SELECT
  HOUR(timestamp) AS hour,
  FLOOR(MINUTE(timestamp) / 15) * 15 AS minute_bucket,
  COUNT(*) AS requests
FROM api_requests
GROUP BY HOUR(timestamp), FLOOR(MINUTE(timestamp) / 15)
ORDER BY hour, minute_bucket;
```

## Finding Peak Hours

```sql
SELECT
  HOUR(order_time) AS hour,
  COUNT(*) AS order_count,
  SUM(total) AS revenue
FROM orders
GROUP BY HOUR(order_time)
ORDER BY order_count DESC
LIMIT 5;
```

## TIME to Seconds Conversion

Convert a TIME value to total seconds:

```sql
SELECT
  duration,
  HOUR(duration) * 3600 + MINUTE(duration) * 60 + SECOND(duration) AS total_seconds
FROM call_logs;
```

Or use the built-in `TIME_TO_SEC()`:

```sql
SELECT TIME_TO_SEC('01:30:45');  -- Returns: 5445
```

## Comparing Time Parts

Find records where the minute equals the second (just an example):

```sql
SELECT *
FROM logs
WHERE MINUTE(log_time) = SECOND(log_time);
```

## Scheduling: Check if Within a Time Window

Find records created within a specific time window each day:

```sql
SELECT *
FROM maintenance_logs
WHERE HOUR(created_at) = 2
  AND MINUTE(created_at) BETWEEN 0 AND 30;
```

## HOUR() with Intervals Greater Than 24

For TIME columns (not DATETIME), HOUR() can return values greater than 23:

```sql
SELECT HOUR('150:30:00');  -- Returns: 150 (for elapsed time tracking)
```

This is useful for tracking cumulative work hours or durations exceeding one day.

## Practical Example: Session Duration Analysis

```sql
SELECT
  user_id,
  HOUR(TIMEDIFF(logout_time, login_time)) AS hours,
  MINUTE(TIMEDIFF(logout_time, login_time)) AS minutes,
  SECOND(TIMEDIFF(logout_time, login_time)) AS seconds
FROM user_sessions
WHERE DATE(login_time) = CURDATE();
```

## Summary

`HOUR()`, `MINUTE()`, and `SECOND()` extract the respective time components from TIME, DATETIME, and TIMESTAMP values in MySQL. Use them for time-based filtering, grouping events by hour or minute bucket, finding peak periods, and converting time to total seconds. Combine with `GROUP BY` for time distribution analysis.
