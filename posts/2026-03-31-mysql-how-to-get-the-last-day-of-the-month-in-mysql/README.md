# How to Get the Last Day of the Month in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Last Day, Date Functions, Month End, Sql

Description: Learn how to use MySQL's LAST_DAY() function to find the last day of a given month, handle month-end billing, and compute month boundaries.

---

## Overview

`LAST_DAY()` is a MySQL date function that returns the last day of the month for a given `DATE` or `DATETIME` value. It automatically accounts for leap years and varying month lengths, making it indispensable for billing cycles, reporting windows, and month-end calculations.

## Basic Syntax

```sql
LAST_DAY(date)
```

Returns a `DATE` value representing the final day of the month containing the input date.

## Basic Examples

```sql
SELECT LAST_DAY('2024-01-15');   -- 2024-01-31
SELECT LAST_DAY('2024-02-01');   -- 2024-02-29 (leap year)
SELECT LAST_DAY('2023-02-01');   -- 2023-02-28 (non-leap year)
SELECT LAST_DAY('2024-06-10');   -- 2024-06-30
SELECT LAST_DAY('2024-12-01');   -- 2024-12-31
SELECT LAST_DAY(NOW());          -- last day of the current month
```

## Computing the First Day of the Month

`LAST_DAY()` makes it easy to also derive the first day:

```sql
-- First day of current month
SELECT DATE_FORMAT(NOW(), '%Y-%m-01') AS first_day;

-- First day of next month
SELECT DATE_ADD(LAST_DAY(NOW()), INTERVAL 1 DAY) AS first_day_next_month;

-- First day of the previous month
SELECT DATE_FORMAT(LAST_DAY(NOW() - INTERVAL 1 MONTH), '%Y-%m-01') AS first_day_prev_month;
```

## Filtering for Month-End Records

```sql
-- All records from the last day of each month
SELECT * FROM transactions
WHERE DAY(txn_date) = DAY(LAST_DAY(txn_date));

-- Equivalent: records where the date equals the month's last day
SELECT * FROM transactions
WHERE txn_date = LAST_DAY(txn_date);
```

## Monthly Reporting Windows

```sql
-- Report for the current month (first day to last day)
SELECT
  SUM(amount) AS monthly_total,
  COUNT(*) AS transaction_count
FROM transactions
WHERE txn_date BETWEEN DATE_FORMAT(NOW(), '%Y-%m-01') AND LAST_DAY(NOW());

-- Report for the previous month
SELECT SUM(amount) AS prev_month_total
FROM transactions
WHERE txn_date BETWEEN
  DATE_FORMAT(LAST_DAY(NOW() - INTERVAL 1 MONTH), '%Y-%m-01') AND
  LAST_DAY(NOW() - INTERVAL 1 MONTH);
```

## Billing Cycle Calculations

```sql
-- Set subscription end date to last day of the current month
UPDATE subscriptions
SET billing_end = LAST_DAY(CURDATE())
WHERE billing_cycle = 'monthly' AND status = 'active';

-- Find subscriptions ending within 3 days
SELECT * FROM subscriptions
WHERE billing_end BETWEEN CURDATE() AND CURDATE() + INTERVAL 3 DAY;
```

## Days Remaining in the Month

```sql
-- How many days remain in the current month
SELECT DATEDIFF(LAST_DAY(CURDATE()), CURDATE()) AS days_remaining;

-- For each order, days remaining when it was placed
SELECT order_id, order_date,
  DATEDIFF(LAST_DAY(order_date), order_date) AS days_left_in_month
FROM orders;
```

## Using LAST_DAY() with QUARTER Boundaries

```sql
-- Last day of the current quarter
SELECT LAST_DAY(
  MAKEDATE(YEAR(CURDATE()), 1) +
  INTERVAL QUARTER(CURDATE()) * 3 - 1 MONTH
) AS last_day_of_quarter;
```

## Summary

`LAST_DAY()` returns the final date of the month for any input date, handling leap years and varying month lengths automatically. It is the foundation for computing month-end billing dates, defining reporting windows, counting days remaining in a month, and deriving first-day-of-next-month values with a simple `LAST_DAY() + INTERVAL 1 DAY` expression.
