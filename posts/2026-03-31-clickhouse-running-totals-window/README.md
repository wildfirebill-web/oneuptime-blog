# How to Compute Running Totals with Window Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, SQL, Time Series, Analytics

Description: Learn how to compute running totals and cumulative sums in ClickHouse using SUM() OVER with frame specifications, with examples for revenue tracking and time series analysis.

---

A running total (also called a cumulative sum) adds up values from the start of a sequence to the current row, producing an ever-increasing (or monotonically changing) aggregate. In ClickHouse, running totals are computed with `SUM()` used as a window function with an appropriate frame specification. This avoids self-joins and correlated subqueries, making the query both simpler and more efficient on large datasets.

## Syntax

```text
SUM(expr) OVER (
    [PARTITION BY partition_expr [, ...]]
    ORDER BY sort_expr [ASC | DESC]
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

The frame `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is the key - it tells ClickHouse to sum all rows from the start of the partition up to and including the current row.

## Basic Running Total

The simplest running total sums revenue across all days in order:

```sql
SELECT
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY sale_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue
FROM daily_sales
ORDER BY sale_date;
```

Each row shows the total revenue accumulated from the first day up to and including that day. This is equivalent to a prefix sum.

## Running Total Per Partition

Add `PARTITION BY` to compute independent running totals for each group. This calculates cumulative revenue per product category:

```sql
SELECT
    category,
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        PARTITION BY category
        ORDER BY sale_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue_by_category
FROM category_daily_sales
ORDER BY category, sale_date;
```

Each category's cumulative total resets to zero at the category boundary. The category with the highest cumulative total on any given date can be found by joining this result back to itself.

## Cumulative Count of Events

Running totals are not limited to revenue. Any additive metric works. This counts cumulative user registrations over time:

```sql
SELECT
    registration_date,
    new_users,
    SUM(new_users) OVER (
        ORDER BY registration_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS total_registered_users
FROM daily_registrations
ORDER BY registration_date;
```

## Running Total as a Percentage of the Grand Total

Divide the running total by the overall total to compute cumulative percentage. This is the basis of Pareto (80/20) analysis:

```sql
SELECT
    product_name,
    revenue,
    SUM(revenue) OVER (
        ORDER BY revenue DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue,
    SUM(revenue) OVER () AS total_revenue,
    ROUND(
        SUM(revenue) OVER (
            ORDER BY revenue DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) / SUM(revenue) OVER () * 100,
        2
    ) AS cumulative_pct
FROM product_sales
WHERE report_month = '2025-12-01'
ORDER BY revenue DESC;
```

`SUM(revenue) OVER ()` with no `ORDER BY` and no frame clause returns the grand total - the sum over all rows in the partition.

## Rolling Sum (Fixed-Window Cumulative)

A rolling sum sums the last N rows rather than all preceding rows. This computes a 7-day rolling revenue total:

```sql
SELECT
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY sale_date ASC
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7d_revenue
FROM daily_sales
ORDER BY sale_date;
```

The first 6 rows will have fewer than 7 rows in their frame (because there are not enough preceding rows), so the rolling sum there is based on however many rows are available.

## Running Total with Multiple Metrics Simultaneously

Compute cumulative sums for several columns in one pass:

```sql
SELECT
    transaction_date,
    gross_sales,
    refunds,
    net_sales,
    SUM(gross_sales) OVER (
        ORDER BY transaction_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_gross,
    SUM(refunds) OVER (
        ORDER BY transaction_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_refunds,
    SUM(net_sales) OVER (
        ORDER BY transaction_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_net
FROM daily_financials
ORDER BY transaction_date;
```

## Resetting Running Totals

Sometimes you need a running total that resets based on a condition - for example, resetting at the start of each fiscal quarter. Use `PARTITION BY` with a computed quarter identifier:

```sql
SELECT
    sale_date,
    daily_revenue,
    toQuarter(sale_date)  AS quarter,
    toYear(sale_date)     AS year,
    SUM(daily_revenue) OVER (
        PARTITION BY toYear(sale_date), toQuarter(sale_date)
        ORDER BY sale_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_quarterly_revenue
FROM daily_sales
ORDER BY sale_date;
```

## Verifying Results with a Self-Join Alternative

For small datasets, it can be useful to verify the window function result against the traditional self-join approach:

```sql
-- Window function approach (efficient)
SELECT
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY sale_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total_window
FROM daily_sales
ORDER BY sale_date;
```

```sql
-- Self-join approach (less efficient, for verification only)
SELECT
    a.sale_date,
    a.daily_revenue,
    SUM(b.daily_revenue) AS running_total_self_join
FROM daily_sales a
JOIN daily_sales b ON b.sale_date <= a.sale_date
GROUP BY a.sale_date, a.daily_revenue
ORDER BY a.sale_date;
```

Both produce the same result. The window function version requires a single table scan; the self-join version requires O(n^2) comparisons.

## Performance Considerations

ClickHouse processes window functions after the `WHERE` and `GROUP BY` phases, so pre-filtering the dataset before the window calculation reduces memory usage:

```sql
-- Filter first, then compute running total
SELECT
    sale_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY sale_date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue
FROM daily_sales
WHERE sale_date BETWEEN '2025-01-01' AND '2025-12-31'
  AND region = 'EMEA'
ORDER BY sale_date;
```

For very large time series stored in `MergeTree` tables, ensure the table's `ORDER BY` key includes the column used in the window's `ORDER BY` to benefit from pre-sorted storage.

## Summary

Running totals in ClickHouse are computed with `SUM() OVER (ORDER BY ... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`. This single-pass approach is dramatically more efficient than self-joins and scales to billions of rows. Use `PARTITION BY` to reset the running total per group, omit the frame clause (or use `OVER ()`) for grand totals, and bound the frame (e.g., `6 PRECEDING AND CURRENT ROW`) for rolling sums. Running totals underpin Pareto analysis, fiscal period tracking, user growth charts, and any report where cumulative progress matters.
