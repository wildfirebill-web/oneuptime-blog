# How to Use DAYOFWEEK() and DAYOFYEAR() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Dayofweek, Dayofyear, Date Functions, Sql

Description: Learn how to use DAYOFWEEK() and DAYOFYEAR() functions in MySQL to extract the day-of-week and day-of-year from date values for filtering and grouping.

---

## Introduction

MySQL provides `DAYOFWEEK()` and `DAYOFYEAR()` as date extraction functions. `DAYOFWEEK()` returns the numeric day of the week (1=Sunday through 7=Saturday), and `DAYOFYEAR()` returns the day number within the year (1 to 366). These are useful for scheduling, trend analysis, and time-based grouping.

## DAYOFWEEK() Syntax

```sql
DAYOFWEEK(date)
```

Returns an integer from 1 (Sunday) to 7 (Saturday). This follows the ODBC standard.

| Return Value | Day |
|-------------|-----|
| 1 | Sunday |
| 2 | Monday |
| 3 | Tuesday |
| 4 | Wednesday |
| 5 | Thursday |
| 6 | Friday |
| 7 | Saturday |

## DAYOFYEAR() Syntax

```sql
DAYOFYEAR(date)
```

Returns an integer from 1 to 366 representing the day within the year.

## Basic DAYOFWEEK() Examples

```sql
SELECT DAYOFWEEK('2024-01-01');  -- Returns: 2  (Monday - Jan 1, 2024)
SELECT DAYOFWEEK('2024-01-07');  -- Returns: 1  (Sunday)
SELECT DAYOFWEEK('2024-01-06');  -- Returns: 7  (Saturday)
SELECT DAYOFWEEK(NOW());         -- Returns: current day of week
```

## Basic DAYOFYEAR() Examples

```sql
SELECT DAYOFYEAR('2024-01-01');  -- Returns: 1
SELECT DAYOFYEAR('2024-02-29');  -- Returns: 60 (2024 is a leap year)
SELECT DAYOFYEAR('2024-12-31'); -- Returns: 366 (leap year)
SELECT DAYOFYEAR('2023-12-31'); -- Returns: 365 (non-leap year)
```

## Filtering by Day of Week

Find all orders placed on weekends:

```sql
SELECT id, customer_id, total_amount, order_date
FROM orders
WHERE DAYOFWEEK(order_date) IN (1, 7);  -- 1=Sunday, 7=Saturday
```

Find weekday orders only:

```sql
SELECT id, order_date, total_amount
FROM orders
WHERE DAYOFWEEK(order_date) BETWEEN 2 AND 6;  -- Mon to Fri
```

## Grouping by Day of Week

Count orders by day of week to find peak days:

```sql
SELECT
  DAYOFWEEK(order_date) AS day_number,
  CASE DAYOFWEEK(order_date)
    WHEN 1 THEN 'Sunday'
    WHEN 2 THEN 'Monday'
    WHEN 3 THEN 'Tuesday'
    WHEN 4 THEN 'Wednesday'
    WHEN 5 THEN 'Thursday'
    WHEN 6 THEN 'Friday'
    WHEN 7 THEN 'Saturday'
  END AS day_name,
  COUNT(*) AS order_count,
  SUM(total_amount) AS total_revenue
FROM orders
GROUP BY DAYOFWEEK(order_date)
ORDER BY day_number;
```

## Using DAYNAME() for Readable Output

`DAYNAME()` returns the name of the day instead of a number:

```sql
SELECT
  DAYNAME(order_date) AS day_name,
  COUNT(*) AS order_count
FROM orders
GROUP BY DAYNAME(order_date), DAYOFWEEK(order_date)
ORDER BY DAYOFWEEK(order_date);
```

## DAYOFYEAR() for Seasonal Analysis

Group events by day of year to find seasonal patterns:

```sql
SELECT
  DAYOFYEAR(sale_date) AS day_of_year,
  SUM(revenue) AS daily_revenue
FROM sales
WHERE YEAR(sale_date) = 2024
GROUP BY DAYOFYEAR(sale_date)
ORDER BY day_of_year;
```

## Finding Records Near a Specific Day of Year

Find records within 7 days of a specific date's day-of-year position (anniversary logic):

```sql
-- Find customers whose birthdays fall within the next 7 days
SELECT name, birth_date
FROM customers
WHERE DAYOFYEAR(DATE_FORMAT(birth_date, CONCAT(YEAR(NOW()), '-%m-%d')))
  BETWEEN DAYOFYEAR(NOW()) AND DAYOFYEAR(NOW()) + 7;
```

## Weekday vs Weekend Sales Comparison

```sql
SELECT
  IF(DAYOFWEEK(sale_date) IN (1, 7), 'Weekend', 'Weekday') AS day_type,
  COUNT(*) AS sale_count,
  AVG(amount) AS avg_amount,
  SUM(amount) AS total_amount
FROM sales
GROUP BY IF(DAYOFWEEK(sale_date) IN (1, 7), 'Weekend', 'Weekday');
```

## DAYOFWEEK vs WEEKDAY()

MySQL also has `WEEKDAY()` which uses a different numbering (0=Monday, 6=Sunday):

```sql
SELECT
  DAYOFWEEK('2024-01-01'),  -- Returns: 2 (Monday, ODBC standard)
  WEEKDAY('2024-01-01');    -- Returns: 0 (Monday, 0-based)
```

Use `DAYOFWEEK` for ODBC compatibility (1=Sunday) and `WEEKDAY` when 0=Monday is more natural.

## Quarter of Year from DAYOFYEAR

```sql
SELECT
  date_col,
  DAYOFYEAR(date_col) AS day_of_year,
  CEIL(DAYOFYEAR(date_col) / 91.25) AS approximate_quarter
FROM events;
-- Better to use QUARTER() for exact quarters
```

## Practical: Scheduling Report

Find tasks due on specific days of the week:

```sql
SELECT
  task_name,
  due_date,
  DAYNAME(due_date) AS due_day,
  CASE
    WHEN DAYOFWEEK(due_date) IN (1, 7) THEN 'Weekend'
    ELSE 'Weekday'
  END AS day_type
FROM tasks
WHERE due_date BETWEEN NOW() AND DATE_ADD(NOW(), INTERVAL 30 DAY)
ORDER BY due_date;
```

## Summary

`DAYOFWEEK()` returns the day of the week as a number (1=Sunday, 7=Saturday) and `DAYOFYEAR()` returns the numeric position of a date within its year (1-366). Use them for weekday/weekend filtering, identifying peak days, seasonal trend analysis, and scheduling logic. Combine with `DAYNAME()` for human-readable labels and `CASE WHEN` for categorizing days.
