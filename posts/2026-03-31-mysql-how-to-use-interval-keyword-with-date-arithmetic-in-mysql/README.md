# How to Use INTERVAL Keyword with Date Arithmetic in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Interval, Date Arithmetic, Date Functions, Sql

Description: Learn how to use MySQL's INTERVAL keyword with DATE_ADD() and DATE_SUB() to add or subtract time units from date and datetime values.

---

## Overview

The `INTERVAL` keyword in MySQL defines a time span that can be added to or subtracted from a `DATE`, `DATETIME`, or `TIMESTAMP` value. Combined with `DATE_ADD()`, `DATE_SUB()`, or the `+` and `-` operators, it enables precise date arithmetic for a wide range of scheduling, filtering, and reporting tasks.

## Basic Syntax

```sql
DATE_ADD(date, INTERVAL value unit)
DATE_SUB(date, INTERVAL value unit)
date + INTERVAL value unit
date - INTERVAL value unit
```

`unit` can be: `SECOND`, `MINUTE`, `HOUR`, `DAY`, `WEEK`, `MONTH`, `QUARTER`, `YEAR`, and compound units like `MINUTE_SECOND`, `HOUR_MINUTE`, `DAY_HOUR`, `YEAR_MONTH`, etc.

## Common INTERVAL Examples

```sql
-- Add days
SELECT DATE_ADD('2024-06-01', INTERVAL 10 DAY);     -- 2024-06-11
SELECT '2024-06-01' + INTERVAL 10 DAY;              -- same result

-- Subtract months
SELECT DATE_SUB('2024-06-01', INTERVAL 3 MONTH);    -- 2024-03-01

-- Add hours and minutes
SELECT DATE_ADD('2024-06-01 08:00:00', INTERVAL 90 MINUTE);  -- 2024-06-01 09:30:00

-- Add weeks
SELECT DATE_ADD('2024-06-01', INTERVAL 2 WEEK);     -- 2024-06-15

-- Add a year and a quarter
SELECT DATE_ADD('2024-01-01', INTERVAL 1 YEAR);     -- 2025-01-01
SELECT DATE_ADD('2024-01-01', INTERVAL 1 QUARTER);  -- 2024-04-01
```

## Using INTERVAL with NOW() and CURDATE()

```sql
-- Last 7 days
SELECT * FROM orders
WHERE order_date >= DATE_SUB(NOW(), INTERVAL 7 DAY);

-- Last 30 days
SELECT * FROM logs
WHERE created_at >= NOW() - INTERVAL 30 DAY;

-- Next 14 days
SELECT * FROM tasks
WHERE due_date BETWEEN CURDATE() AND CURDATE() + INTERVAL 14 DAY;

-- Records from the last 6 months
SELECT * FROM subscriptions
WHERE start_date >= DATE_SUB(CURDATE(), INTERVAL 6 MONTH);
```

## Compound Interval Units

```sql
-- YEAR_MONTH: 'years-months'
SELECT DATE_ADD('2024-01-15', INTERVAL '1-6' YEAR_MONTH);  -- 2025-07-15

-- DAY_HOUR: 'days hours'
SELECT DATE_ADD('2024-06-01 00:00:00', INTERVAL '2 12' DAY_HOUR);  -- 2024-06-03 12:00:00

-- HOUR_MINUTE: 'hours:minutes'
SELECT DATE_ADD('2024-06-01 08:00:00', INTERVAL '1:30' HOUR_MINUTE);  -- 2024-06-01 09:30:00

-- DAY_SECOND: 'days hours:minutes:seconds'
SELECT DATE_ADD('2024-06-01 00:00:00', INTERVAL '1 2:30:00' DAY_SECOND);  -- 2024-06-02 02:30:00
```

## Dynamic INTERVAL Values from Columns

```sql
-- Extend each subscription by its term_months
UPDATE subscriptions
SET expiry_date = DATE_ADD(start_date, INTERVAL term_months MONTH);

-- Schedule reminders N days before due date
SELECT task_name,
  due_date - INTERVAL reminder_days DAY AS remind_on
FROM tasks
WHERE due_date > CURDATE();
```

## INTERVAL in Scheduled Event Definitions

```sql
CREATE EVENT purge_old_logs
ON SCHEDULE EVERY 1 DAY
DO
  DELETE FROM logs WHERE created_at < NOW() - INTERVAL 90 DAY;
```

## Practical Example: Generating a Date Range

```sql
-- Generate a list of dates for the next 7 days using INTERVAL
SELECT CURDATE() + INTERVAL n DAY AS future_date
FROM (
  SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2
  UNION ALL SELECT 3 UNION ALL SELECT 4
  UNION ALL SELECT 5 UNION ALL SELECT 6
) nums;
```

## End-of-Month Handling

```sql
-- MySQL handles month-end overflow automatically
SELECT DATE_ADD('2024-01-31', INTERVAL 1 MONTH);  -- 2024-02-29 (leap year)
SELECT DATE_ADD('2024-03-31', INTERVAL 1 MONTH);  -- 2024-04-30
```

## Summary

The `INTERVAL` keyword is the core building block for date arithmetic in MySQL, supporting units from seconds through years and compound units. Use it with `DATE_ADD()`, `DATE_SUB()`, or the `+` and `-` operators to shift dates and filter within time windows. MySQL handles month-end overflow automatically, so adding months to March 31 correctly rounds to April 30.
