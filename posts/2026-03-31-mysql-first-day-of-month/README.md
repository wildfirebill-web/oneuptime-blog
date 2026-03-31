# How to Get the First Day of the Month in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Database, Query

Description: Learn how to get the first day of the current or any given month in MySQL using DATE_FORMAT, DATE_SUB, and other date arithmetic techniques.

---

## Standard Technique Using DATE_FORMAT

The most readable way to get the first day of the month for any date is to use `DATE_FORMAT()` to zero out the day component:

```sql
SELECT DATE_FORMAT(CURDATE(), '%Y-%m-01') AS first_day;
-- Example: 2026-03-01
```

The format `%Y-%m-01` keeps the year and month but hard-codes the day to `01`. The result is a string, so cast it to `DATE` if you need date arithmetic:

```sql
SELECT CAST(DATE_FORMAT(CURDATE(), '%Y-%m-01') AS DATE) AS first_day;
```

## Alternative: DATE_SUB with DAY()

Subtract the current day-of-month minus 1 from the current date:

```sql
SELECT DATE_SUB(CURDATE(), INTERVAL DAY(CURDATE()) - 1 DAY) AS first_day;
-- 2026-03-01
```

This works by computing how many days to roll back to reach the first. `DAY(CURDATE())` returns the current day number (e.g., 31 on March 31), so subtracting 30 days lands on March 1.

## First Day of Any Given Month

Replace `CURDATE()` with any date column or literal:

```sql
SELECT
  order_date,
  DATE_SUB(order_date, INTERVAL DAY(order_date) - 1 DAY) AS month_start
FROM orders
LIMIT 5;
```

Or using `DATE_FORMAT`:

```sql
SELECT
  invoice_date,
  CAST(DATE_FORMAT(invoice_date, '%Y-%m-01') AS DATE) AS month_start
FROM invoices;
```

## First Day of the Next Month

```sql
SELECT DATE_FORMAT(CURDATE() + INTERVAL 1 MONTH, '%Y-%m-01') AS next_month_start;
-- 2026-04-01
```

## First Day of the Previous Month

```sql
SELECT DATE_FORMAT(CURDATE() - INTERVAL 1 MONTH, '%Y-%m-01') AS prev_month_start;
-- 2026-02-01
```

## Practical Use Case: Month-to-Date Filtering

A common pattern is filtering rows from the beginning of the current month to today:

```sql
SELECT
  order_id,
  order_date,
  amount
FROM orders
WHERE order_date >= CAST(DATE_FORMAT(CURDATE(), '%Y-%m-01') AS DATE)
  AND order_date <= CURDATE();
```

This avoids hard-coding dates and works correctly even when crossing year boundaries.

## Generating a Monthly Summary with GROUP BY

Aggregate sales by month using the first day as the group key:

```sql
SELECT
  DATE_SUB(order_date, INTERVAL DAY(order_date) - 1 DAY) AS month_start,
  SUM(amount)  AS total_sales,
  COUNT(*)     AS order_count
FROM orders
GROUP BY month_start
ORDER BY month_start;
```

## First Day of a Specific Year and Month

If you have year and month as separate integers (e.g., from a report parameter):

```sql
SELECT STR_TO_DATE(CONCAT('2026', '-', '03', '-01'), '%Y-%m-%d') AS first_day;
-- 2026-03-01
```

Or using `MAKEDATE()` with the month offset:

```sql
-- First day of March 2026 = day 60 of 2026
SELECT MAKEDATE(2026, 60) AS march_first;
-- 2026-03-01
```

This approach is less intuitive, so `STR_TO_DATE(CONCAT(...), ...)` or `DATE_FORMAT` is preferred.

## Summary

The cleanest way to get the first day of the month in MySQL is `DATE_FORMAT(date, '%Y-%m-01')` cast to `DATE`, or `DATE_SUB(date, INTERVAL DAY(date) - 1 DAY)`. Both handle year rollovers correctly. Use these patterns for month-to-date filtering, monthly aggregations, and date range boundaries in reports and dashboards.
