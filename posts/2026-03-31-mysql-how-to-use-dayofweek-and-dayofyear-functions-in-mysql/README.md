# How to Use DAYOFWEEK() and DAYOFYEAR() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Dayofweek, Dayofyear, Date Function, SQL

Description: Learn how to use MySQL's DAYOFWEEK() and DAYOFYEAR() functions to extract the day-of-week index and day-of-year number from date values.

---

## Overview

MySQL provides two handy functions for extracting day-position information from dates: `DAYOFWEEK()` returns a number from 1 (Sunday) to 7 (Saturday), and `DAYOFYEAR()` returns the day's ordinal position within the year (1 to 366).

## DAYOFWEEK() Syntax and Return Values

```sql
DAYOFWEEK(date)
```

Returns an integer where 1=Sunday, 2=Monday, ..., 7=Saturday. This follows the ODBC standard.

```sql
SELECT DAYOFWEEK('2024-01-01');  -- 2 (Monday)
SELECT DAYOFWEEK('2024-01-07');  -- 1 (Sunday)
SELECT DAYOFWEEK(NOW());         -- current day of week
```

## DAYOFYEAR() Syntax and Return Values

```sql
DAYOFYEAR(date)
```

Returns an integer from 1 to 366 representing the ordinal day within the year.

```sql
SELECT DAYOFYEAR('2024-01-01');  -- 1
SELECT DAYOFYEAR('2024-12-31');  -- 366 (2024 is a leap year)
SELECT DAYOFYEAR('2023-12-31');  -- 365
SELECT DAYOFYEAR(NOW());         -- current day of year
```

## Filtering by Day of Week

```sql
-- Get all orders placed on weekends (Saturday=7, Sunday=1)
SELECT * FROM orders
WHERE DAYOFWEEK(order_date) IN (1, 7);

-- Get orders placed on weekdays
SELECT * FROM orders
WHERE DAYOFWEEK(order_date) BETWEEN 2 AND 6;

-- Count orders by day of week
SELECT DAYOFWEEK(order_date) AS dow,
  CASE DAYOFWEEK(order_date)
    WHEN 1 THEN 'Sunday'
    WHEN 2 THEN 'Monday'
    WHEN 3 THEN 'Tuesday'
    WHEN 4 THEN 'Wednesday'
    WHEN 5 THEN 'Thursday'
    WHEN 6 THEN 'Friday'
    WHEN 7 THEN 'Saturday'
  END AS day_name,
  COUNT(*) AS total_orders
FROM orders
GROUP BY DAYOFWEEK(order_date)
ORDER BY dow;
```

## Filtering by Day of Year

```sql
-- Find all events in the first quarter (days 1-91)
SELECT * FROM events
WHERE DAYOFYEAR(event_date) BETWEEN 1 AND 91;

-- Find records matching today's day-of-year across all years
SELECT * FROM logs
WHERE DAYOFYEAR(log_date) = DAYOFYEAR(CURDATE());

-- Compare day-of-year progress between years
SELECT
  YEAR(sale_date) AS yr,
  DAYOFYEAR(sale_date) AS doy,
  SUM(amount) AS daily_sales
FROM sales
WHERE YEAR(sale_date) IN (2023, 2024)
GROUP BY yr, doy
ORDER BY doy, yr;
```

## Combining DAYOFWEEK() and DAYOFYEAR()

```sql
-- Identify which calendar weeks had weekend orders
SELECT
  WEEK(order_date) AS week_num,
  SUM(CASE WHEN DAYOFWEEK(order_date) IN (1,7) THEN 1 ELSE 0 END) AS weekend_orders,
  SUM(CASE WHEN DAYOFWEEK(order_date) BETWEEN 2 AND 6 THEN 1 ELSE 0 END) AS weekday_orders
FROM orders
WHERE YEAR(order_date) = 2024
GROUP BY WEEK(order_date);
```

## Using DAYOFWEEK() for Scheduling Logic

```sql
-- Check if today is a business day
SELECT
  CASE WHEN DAYOFWEEK(CURDATE()) IN (1, 7)
    THEN 'Weekend'
    ELSE 'Weekday'
  END AS day_type;

-- Find the next Monday from a given date
SELECT
  DATE_ADD('2024-06-15', INTERVAL (9 - DAYOFWEEK('2024-06-15')) % 7 DAY) AS next_monday;
```

## Performance Note

Both functions are not index-friendly when used in a `WHERE` clause on an indexed date column because MySQL cannot use the index to resolve the function output. For high-performance queries, consider adding a generated column:

```sql
ALTER TABLE orders
  ADD COLUMN order_dow TINYINT AS (DAYOFWEEK(order_date)) STORED,
  ADD INDEX idx_order_dow (order_dow);
```

## Summary

`DAYOFWEEK()` returns a 1-7 value (Sunday=1) and `DAYOFYEAR()` returns the ordinal day within the year (1-366). Both are useful for reporting, scheduling, and filtering by time patterns. For frequent filtering on these values, consider adding stored generated columns with indexes to maintain query performance.
