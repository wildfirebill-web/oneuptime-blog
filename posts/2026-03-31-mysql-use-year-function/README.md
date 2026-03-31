# How to Use YEAR() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Database, Query

Description: Learn how to use MySQL YEAR() to extract the year from DATE, DATETIME, and TIMESTAMP values for filtering, grouping, and reporting by year.

---

## What Is the YEAR() Function?

The `YEAR()` function in MySQL extracts the year component from a date or datetime value. It returns an integer between 1000 and 9999, or 0 for zero dates.

**Syntax:**

```sql
YEAR(date)
```

The `date` argument can be a `DATE`, `DATETIME`, `TIMESTAMP`, or a string that can be interpreted as a date.

## Basic Usage

```sql
SELECT YEAR('2024-06-15');
-- Returns: 2024

SELECT YEAR(NOW());
-- Returns: current year, e.g. 2026

SELECT YEAR('2024-06-15 14:30:00');
-- Returns: 2024
```

`YEAR()` returns `NULL` if the argument is `NULL`, and returns `0` for invalid dates like `'0000-00-00'`.

## Filtering Rows by Year

A common use case is filtering records that belong to a specific year:

```sql
SELECT order_id, customer_id, created_at
FROM orders
WHERE YEAR(created_at) = 2024;
```

For large tables, this pattern prevents index use on `created_at`. A range condition is more index-friendly:

```sql
SELECT order_id, customer_id, created_at
FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at < '2025-01-01';
```

Both return the same rows, but the range form uses an index on `created_at` efficiently.

## Grouping Data by Year

Use `YEAR()` in a `GROUP BY` clause to aggregate data per year:

```sql
SELECT
  YEAR(order_date) AS order_year,
  COUNT(*)         AS total_orders,
  SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date)
ORDER BY order_year;
```

Sample output:

| order_year | total_orders | revenue   |
|------------|--------------|-----------|
| 2022       | 1240         | 182000.00 |
| 2023       | 1890         | 267000.00 |
| 2024       | 2150         | 310000.00 |

## Combining YEAR() with Other Date Functions

You can combine `YEAR()` with `MONTH()` or `WEEK()` for finer groupings:

```sql
-- Group by year and month
SELECT
  YEAR(sale_date)  AS yr,
  MONTH(sale_date) AS mo,
  SUM(amount)      AS monthly_revenue
FROM sales
GROUP BY YEAR(sale_date), MONTH(sale_date)
ORDER BY yr, mo;
```

To get a formatted label, combine with `DATE_FORMAT()`:

```sql
SELECT
  DATE_FORMAT(sale_date, '%Y-%m') AS period,
  SUM(amount) AS revenue
FROM sales
GROUP BY period
ORDER BY period;
```

## Updating Rows Based on Year

`YEAR()` can be used in `UPDATE` statements:

```sql
UPDATE subscriptions
SET plan_type = 'legacy'
WHERE YEAR(created_at) < 2020;
```

## Extracting Year from a String Column

If dates are stored as strings, MySQL still accepts them in `YEAR()` as long as they follow a recognizable format:

```sql
SELECT YEAR('2023/11/05');
-- Returns: 2023
```

For non-standard formats, convert first with `STR_TO_DATE()`:

```sql
SELECT YEAR(STR_TO_DATE('05-11-2023', '%d-%m-%Y'));
-- Returns: 2023
```

## Checking the Current Year

```sql
SELECT YEAR(CURDATE()) AS this_year;
-- Returns: 2026 (current year)
```

You can use this dynamically in queries:

```sql
SELECT *
FROM events
WHERE YEAR(event_date) = YEAR(CURDATE());
```

## Summary

`YEAR()` extracts the four-digit year from any date-like value in MySQL. It is most useful for filtering and grouping queries by year, building year-over-year reports, and combining with `MONTH()` for period analysis. For performance on indexed columns, prefer range comparisons over wrapping the column in `YEAR()`.
