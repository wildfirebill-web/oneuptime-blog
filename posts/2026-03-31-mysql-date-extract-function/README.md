# How to Use DATE() Function to Extract Date in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Database

Description: Learn how MySQL's DATE() function extracts the date portion from a DATETIME or TIMESTAMP value, enabling date-level grouping and filtering in queries.

---

## What is DATE()?

The `DATE()` function in MySQL extracts the date part (year, month, and day) from a `DATETIME`, `TIMESTAMP`, or string expression, returning a `DATE` value in `YYYY-MM-DD` format. It is most commonly used to truncate a datetime to the day level for grouping, filtering, and comparisons without considering the time component.

The syntax is:

```sql
DATE(expr)
```

`expr` can be a `DATETIME`, `TIMESTAMP`, or a string in a recognizable datetime format.

## Basic Examples

```sql
SELECT DATE('2026-03-31 14:22:07');
-- Result: 2026-03-31

SELECT DATE(NOW());
-- Result: 2026-03-31  (current date)

SELECT DATE('2026-03-31');
-- Result: 2026-03-31
```

## Filtering by Date (Ignoring Time)

A very common use case is filtering a `DATETIME` column by a specific date, ignoring the time portion:

```sql
SELECT *
FROM orders
WHERE DATE(created_at) = '2026-03-31';
```

Note: wrapping a column in `DATE()` prevents index usage on `created_at`. For large tables, use a range comparison instead:

```sql
-- Index-friendly alternative
SELECT *
FROM orders
WHERE created_at >= '2026-03-31 00:00:00'
  AND created_at <  '2026-04-01 00:00:00';
```

## Daily Aggregation

Group events or transactions by day:

```sql
SELECT
  DATE(created_at) AS order_date,
  COUNT(*)          AS order_count,
  SUM(total)        AS daily_revenue
FROM orders
GROUP BY DATE(created_at)
ORDER BY order_date DESC;
```

## Comparing Dates

Find all records where the date portion matches today:

```sql
SELECT *
FROM audit_log
WHERE DATE(logged_at) = CURDATE();
```

Find records from yesterday:

```sql
SELECT *
FROM activity
WHERE DATE(timestamp) = CURDATE() - INTERVAL 1 DAY;
```

## Extracting Date for Joins

Join two tables on the date portion of datetime columns:

```sql
SELECT
  s.sale_date,
  s.amount,
  t.target_revenue
FROM sales s
JOIN daily_targets t ON DATE(s.created_at) = t.sale_date;
```

## Using DATE() with String Inputs

MySQL can parse date strings automatically:

```sql
SELECT DATE('March 31, 2026');
-- Result: NULL  (format not recognized)

SELECT DATE('2026-03-31T14:22:07');
-- Result: 2026-03-31

SELECT DATE('20260331142207');
-- Result: 2026-03-31
```

## Truncating Timestamps for Bucketing

Group records by week using `DATE()` with `WEEK()`:

```sql
SELECT
  WEEK(DATE(created_at), 1) AS iso_week,
  COUNT(*) AS events
FROM user_events
GROUP BY WEEK(DATE(created_at), 1);
```

## Computing Date Differences

Calculate how many days ago each record was created:

```sql
SELECT
  id,
  created_at,
  DATEDIFF(CURDATE(), DATE(created_at)) AS days_ago
FROM posts
ORDER BY created_at DESC;
```

## DATE() vs DATE_FORMAT() vs EXTRACT()

```sql
-- DATE() returns YYYY-MM-DD
SELECT DATE(NOW());                       -- 2026-03-31

-- DATE_FORMAT() returns a formatted string
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d');    -- 2026-03-31

-- EXTRACT() returns a numeric part
SELECT EXTRACT(DAY FROM NOW());           -- 31
```

Use `DATE()` when you need a `DATE` value for comparison. Use `DATE_FORMAT()` for display strings. Use `EXTRACT()` for numeric parts.

## Summary

`DATE()` extracts the `YYYY-MM-DD` date portion from a `DATETIME` or `TIMESTAMP` value. It is primarily used for date-level filtering, daily aggregation with `GROUP BY DATE(column)`, and date comparisons that ignore time. For index performance on large tables, prefer explicit range predicates (`>= date AND < next_date`) over wrapping a column in `DATE()`. Use `CURDATE()` as a shorthand for `DATE(NOW())`.
