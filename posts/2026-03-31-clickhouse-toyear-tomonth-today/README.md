# How to Use toYear(), toMonth(), toDayOfMonth() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Analytics, Aggregation, Query

Description: Learn how to extract year, month, and day components from DateTime values in ClickHouse using toYear(), toMonth(), and toDayOfMonth().

---

Extracting individual components from a date or timestamp is one of the most frequent operations in analytical SQL. ClickHouse provides dedicated functions for each component: `toYear()`, `toMonth()`, and `toDayOfMonth()`. These functions accept both `Date` and `DateTime` values and return integers that can be used in `WHERE` clauses, `GROUP BY` expressions, or computed columns.

This post explains each function in detail and shows how to combine them for grouping, filtering, and building date-based aggregations.

## toYear() - Extract the Year

`toYear(dt)` returns a `UInt16` representing the four-digit year of the given date or datetime.

```sql
SELECT toYear(toDate('2026-03-31')) AS year;
```

```text
year
-----
2026
```

Use `toYear()` in a `GROUP BY` to aggregate data by calendar year:

```sql
SELECT
    toYear(order_date)  AS year,
    count()             AS total_orders,
    sum(amount)         AS total_revenue
FROM orders
GROUP BY year
ORDER BY year;
```

Filter for a specific year without hardcoding a date range:

```sql
-- All events from 2025
SELECT event_id, event_name, created_at
FROM events
WHERE toYear(created_at) = 2025
ORDER BY created_at;
```

Year-over-year comparison using conditional aggregation:

```sql
SELECT
    toYear(sale_date)             AS year,
    sum(revenue)                  AS total_revenue,
    lagInFrame(sum(revenue)) OVER (ORDER BY toYear(sale_date)) AS prev_year_revenue
FROM sales
GROUP BY year
ORDER BY year;
```

## toMonth() - Extract the Month

`toMonth(dt)` returns a `UInt8` in the range 1 to 12 representing the calendar month.

```sql
SELECT toMonth(toDate('2026-03-31')) AS month;
```

```text
month
------
3
```

Group sales by month to identify seasonal patterns:

```sql
SELECT
    toMonth(sale_date)  AS month,
    count()             AS orders,
    sum(revenue)        AS revenue
FROM sales
WHERE toYear(sale_date) = 2025
GROUP BY month
ORDER BY month;
```

Filter for a specific month across all years - useful for detecting seasonality:

```sql
-- All Decembers across every year in the table
SELECT
    toYear(sale_date)  AS year,
    sum(revenue)       AS december_revenue
FROM sales
WHERE toMonth(sale_date) = 12
GROUP BY year
ORDER BY year;
```

## toDayOfMonth() - Extract the Day

`toDayOfMonth(dt)` returns a `UInt8` in the range 1 to 31 representing the day within the month.

```sql
SELECT toDayOfMonth(toDate('2026-03-31')) AS day;
```

```text
day
----
31
```

Identify which days of the month have the highest transaction volume:

```sql
SELECT
    toDayOfMonth(transaction_date)  AS day_of_month,
    count()                         AS transactions,
    sum(amount)                     AS total_amount
FROM transactions
GROUP BY day_of_month
ORDER BY day_of_month;
```

This query is useful for spotting patterns like salary-day spending spikes on the 1st and 15th of each month.

## Combining All Three for Precise Filtering

When you need to match a specific year, month, and day without constructing a full date string, combine the three functions:

```sql
SELECT *
FROM events
WHERE toYear(created_at)       = 2026
  AND toMonth(created_at)      = 3
  AND toDayOfMonth(created_at) = 31;
```

This is equivalent to `WHERE toDate(created_at) = '2026-03-31'` but can be useful when the components come from variables or when you want to reuse them in `SELECT` as well.

## Building Year-Month Aggregations

A common reporting pattern groups data by year and month together. Combine `toYear()` and `toMonth()` into a composite key:

```sql
SELECT
    toYear(event_date)  AS year,
    toMonth(event_date) AS month,
    count()             AS events,
    uniq(user_id)       AS unique_users
FROM page_views
GROUP BY year, month
ORDER BY year, month;
```

For a more display-friendly label, format it as a string:

```sql
SELECT
    concat(
        toString(toYear(event_date)), '-',
        leftPad(toString(toMonth(event_date)), 2, '0')
    ) AS year_month,
    count() AS page_views
FROM page_views
GROUP BY year_month
ORDER BY year_month;
```

## Filtering by Month and Day Across Years (Anniversary Queries)

To find all events that occurred on a specific month-day combination across multiple years (for example, to compare March 31st performance year over year):

```sql
SELECT
    toYear(sale_date)  AS year,
    sum(revenue)       AS revenue_on_march_31
FROM sales
WHERE toMonth(sale_date) = 3
  AND toDayOfMonth(sale_date) = 31
GROUP BY year
ORDER BY year;
```

## Summary

`toYear()`, `toMonth()`, and `toDayOfMonth()` are the building blocks of component-level date analysis in ClickHouse. Use them individually when filtering or grouping on a single component, and combine them for precise multi-component filtering. They work equally well on `Date` and `DateTime` columns, making them versatile tools for any time-series aggregation or cohort analysis.
