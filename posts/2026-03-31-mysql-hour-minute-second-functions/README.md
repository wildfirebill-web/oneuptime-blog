# How to Use HOUR(), MINUTE(), SECOND() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Time Function, Database

Description: Learn how to use MySQL HOUR(), MINUTE(), and SECOND() to extract time components from DATETIME and TIME values for time-based filtering and analysis.

---

## Overview

MySQL provides three functions to extract individual time components from `TIME`, `DATETIME`, and `TIMESTAMP` values:

- `HOUR(time)` - returns the hour component (0-23 for TIME, 0-838 for large TIME values)
- `MINUTE(time)` - returns the minute component (0-59)
- `SECOND(time)` - returns the second component (0-59)

**Syntax:**

```sql
HOUR(time)
MINUTE(time)
SECOND(time)
```

Each function accepts `TIME`, `DATETIME`, `TIMESTAMP`, or a string parseable as a time value.

## Basic Examples

```sql
SELECT HOUR('14:30:45');
-- Returns: 14

SELECT MINUTE('14:30:45');
-- Returns: 30

SELECT SECOND('14:30:45');
-- Returns: 45

SELECT HOUR(NOW()), MINUTE(NOW()), SECOND(NOW());
-- Returns: current hour, minute, second

SELECT HOUR('2024-06-15 09:05:03');
-- Returns: 9
```

## Filtering by Hour

Find all orders placed during business hours (9 AM to 5 PM):

```sql
SELECT order_id, created_at, total_amount
FROM orders
WHERE HOUR(created_at) BETWEEN 9 AND 17;
```

Find late-night activity (after 10 PM):

```sql
SELECT user_id, action, logged_at
FROM user_activity
WHERE HOUR(logged_at) >= 22;
```

## Filtering by Minute

Find events that occurred in the first 15 minutes of any hour:

```sql
SELECT *
FROM events
WHERE MINUTE(event_time) < 15;
```

Find records logged on the hour (minute = 0):

```sql
SELECT *
FROM metrics
WHERE MINUTE(recorded_at) = 0;
```

## Grouping by Hour

Analyze order volume by hour of day to identify peak times:

```sql
SELECT
  HOUR(created_at) AS hour_of_day,
  COUNT(*)         AS order_count,
  SUM(total_amount) AS revenue
FROM orders
GROUP BY HOUR(created_at)
ORDER BY hour_of_day;
```

This is useful for staffing decisions, infrastructure scaling, and identifying demand patterns.

## Combining All Three Functions

Extract full time components for display:

```sql
SELECT
  order_id,
  CONCAT(
    LPAD(HOUR(created_at), 2, '0'), ':',
    LPAD(MINUTE(created_at), 2, '0'), ':',
    LPAD(SECOND(created_at), 2, '0')
  ) AS order_time
FROM orders
LIMIT 10;
```

## Working with TIME Columns

These functions work directly on `TIME` columns:

```sql
CREATE TABLE shifts (
  shift_id INT PRIMARY KEY,
  start_time TIME,
  end_time TIME
);

SELECT
  shift_id,
  HOUR(start_time) AS start_hour,
  HOUR(end_time)   AS end_hour,
  (HOUR(end_time) - HOUR(start_time)) AS duration_hours
FROM shifts;
```

## Hourly Bucketing

Group records into 4-hour time blocks:

```sql
SELECT
  FLOOR(HOUR(created_at) / 4) * 4 AS hour_block,
  COUNT(*) AS record_count
FROM events
GROUP BY hour_block
ORDER BY hour_block;
```

This produces blocks: 0, 4, 8, 12, 16, 20.

## Performance Note

Wrapping an indexed datetime column in `HOUR()` prevents index use. For high-traffic tables, consider storing the hour separately or using range conditions:

```sql
-- Index-friendly: filter a specific hour today
WHERE created_at >= '2024-06-15 14:00:00'
  AND created_at <  '2024-06-15 15:00:00'
```

## NULL Handling

```sql
SELECT HOUR(NULL), MINUTE(NULL), SECOND(NULL);
-- Returns: NULL, NULL, NULL
```

## Summary

`HOUR()`, `MINUTE()`, and `SECOND()` extract time parts from MySQL date/time values. Use them to filter records by time of day, group events into hourly buckets, and build time-based dashboards. For performance on large tables with indexed datetime columns, prefer range conditions over wrapping columns in these functions.
