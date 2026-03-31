# How to Use SEC_TO_TIME() and TIME_TO_SEC() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Sec To Time, Time To Sec, Time Functions, Sql

Description: Learn how to use MySQL's TIME_TO_SEC() and SEC_TO_TIME() functions to convert between TIME values and total seconds for arithmetic operations.

---

## Overview

MySQL's `TIME_TO_SEC()` converts a `TIME` value into total seconds, and `SEC_TO_TIME()` converts a seconds integer back into a `TIME` value. Together they enable straightforward time arithmetic, averaging durations, and summing time intervals stored as `TIME` columns.

## Basic Syntax

```sql
TIME_TO_SEC(time)   -- returns total seconds as an integer
SEC_TO_TIME(seconds) -- returns a TIME value (HH:MM:SS)
```

## Basic Examples

```sql
-- Convert TIME to seconds
SELECT TIME_TO_SEC('01:00:00');   -- 3600
SELECT TIME_TO_SEC('00:30:00');   -- 1800
SELECT TIME_TO_SEC('23:59:59');   -- 86399
SELECT TIME_TO_SEC('10:15:30');   -- 36930

-- Convert seconds to TIME
SELECT SEC_TO_TIME(3600);         -- 01:00:00
SELECT SEC_TO_TIME(1800);         -- 00:30:00
SELECT SEC_TO_TIME(86399);        -- 23:59:59
SELECT SEC_TO_TIME(90061);        -- 25:01:01 (TIME can exceed 24h)
```

## Time Arithmetic Using Seconds

```sql
-- Add 45 minutes to a time value
SELECT SEC_TO_TIME(TIME_TO_SEC('08:00:00') + 45 * 60) AS result;  -- 08:45:00

-- Subtract 1.5 hours from a time value
SELECT SEC_TO_TIME(TIME_TO_SEC('12:00:00') - TIME_TO_SEC('01:30:00')) AS result;  -- 10:30:00

-- Find the midpoint between two times
SELECT SEC_TO_TIME(
  (TIME_TO_SEC('08:00:00') + TIME_TO_SEC('17:00:00')) / 2
) AS midpoint;  -- 12:30:00
```

## Summing and Averaging TIME Columns

```sql
CREATE TABLE work_logs (
  id INT AUTO_INCREMENT PRIMARY KEY,
  employee_id INT,
  log_date DATE,
  hours_worked TIME
);

INSERT INTO work_logs (employee_id, log_date, hours_worked) VALUES
(1, '2024-06-10', '08:30:00'),
(1, '2024-06-11', '07:45:00'),
(1, '2024-06-12', '09:15:00');

-- Total hours worked per employee
SELECT
  employee_id,
  SEC_TO_TIME(SUM(TIME_TO_SEC(hours_worked))) AS total_hours
FROM work_logs
GROUP BY employee_id;

-- Average hours worked per employee per day
SELECT
  employee_id,
  SEC_TO_TIME(AVG(TIME_TO_SEC(hours_worked))) AS avg_hours
FROM work_logs
GROUP BY employee_id;
```

## Duration Comparisons

```sql
-- Find sessions longer than 2 hours
SELECT * FROM sessions
WHERE TIME_TO_SEC(duration) > TIME_TO_SEC('02:00:00');

-- Equivalent using seconds directly
SELECT * FROM sessions
WHERE TIME_TO_SEC(duration) > 7200;

-- Sort by duration
SELECT * FROM tasks
ORDER BY TIME_TO_SEC(estimated_time) DESC;
```

## Practical Example: Calculating Overtime

```sql
-- Standard workday is 8 hours = 28800 seconds
SELECT
  employee_id,
  log_date,
  hours_worked,
  CASE
    WHEN TIME_TO_SEC(hours_worked) > 28800
      THEN SEC_TO_TIME(TIME_TO_SEC(hours_worked) - 28800)
    ELSE '00:00:00'
  END AS overtime
FROM work_logs;
```

## Converting Between Different Time Representations

```sql
-- Convert decimal hours to TIME
-- 1.75 hours = 1h 45m
SELECT SEC_TO_TIME(1.75 * 3600) AS time_value;  -- 01:45:00

-- Convert TIME to decimal hours
SELECT TIME_TO_SEC('01:45:00') / 3600 AS decimal_hours;  -- 1.75

-- Convert TIME to total minutes
SELECT TIME_TO_SEC('02:30:00') / 60 AS total_minutes;  -- 150
```

## Summary

`TIME_TO_SEC()` and `SEC_TO_TIME()` are complementary functions that bridge MySQL `TIME` values and integer seconds. Use `TIME_TO_SEC()` before applying aggregate functions like `SUM()` or `AVG()` on `TIME` columns, then wrap the result in `SEC_TO_TIME()` to get back a readable time value. `SEC_TO_TIME()` supports values beyond 24 hours, making it suitable for duration tracking.
