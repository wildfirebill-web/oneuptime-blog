# How to Use LAG() Window Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, SQL, Time Series, Analytics

Description: Learn how LAG() accesses values from previous rows within a partition in ClickHouse, with examples for computing deltas, period-over-period comparisons, and trend detection.

---

`LAG()` is a window function that lets you access a value from a row that comes before the current row in the window's ordering. It eliminates the need for self-joins when computing differences between consecutive records - a pattern that appears constantly in time series analysis, financial data, and event tracking. ClickHouse fully supports `LAG()` as part of its window function implementation.

## Syntax

```text
LAG(expr [, offset [, default_value]])
OVER (
    [PARTITION BY partition_expr [, ...]]
    ORDER BY sort_expr [ASC | DESC] [, ...]
)
```

- `expr` - the column or expression whose previous value you want to retrieve.
- `offset` - how many rows back to look (default is 1, meaning the immediately preceding row).
- `default_value` - the value to return when the offset goes beyond the partition boundary (default is `NULL`).

## Basic Usage: Accessing the Previous Row

The simplest application fetches the value from the row immediately before the current one. This query shows each day's revenue alongside the previous day's revenue:

```sql
SELECT
    sale_date,
    revenue,
    LAG(revenue) OVER (ORDER BY sale_date ASC) AS prev_day_revenue
FROM daily_sales
ORDER BY sale_date;
```

The first row (earliest date) has no preceding row, so `prev_day_revenue` is `NULL` for that row.

## Computing Day-Over-Day Deltas

Subtract the lagged value from the current value to get the absolute change, and divide to get the percentage change:

```sql
SELECT
    sale_date,
    revenue,
    LAG(revenue) OVER (ORDER BY sale_date ASC)            AS prev_revenue,
    revenue - LAG(revenue) OVER (ORDER BY sale_date ASC)  AS revenue_delta,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY sale_date ASC))
        / LAG(revenue) OVER (ORDER BY sale_date ASC) * 100,
        2
    )                                                      AS pct_change
FROM daily_sales
ORDER BY sale_date;
```

The `NULL` in the first row propagates through arithmetic expressions, so `revenue_delta` and `pct_change` are also `NULL` for that row. This is generally the desired behavior, but you can replace it with 0 using the default value parameter.

## Using a Default Value Instead of NULL

Pass a third argument to `LAG()` to substitute a value when no preceding row exists:

```sql
SELECT
    sale_date,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY sale_date ASC) AS prev_revenue
FROM daily_sales
ORDER BY sale_date;
```

Now the first row shows `prev_revenue = 0` instead of `NULL`. This is useful when downstream calculations cannot handle `NULL` values.

## Offset Greater Than 1: Comparing to N Periods Ago

Set the offset to any positive integer to look back multiple rows. This computes week-over-week change by looking back 7 rows in a daily series:

```sql
SELECT
    sale_date,
    revenue,
    LAG(revenue, 7, 0) OVER (ORDER BY sale_date ASC) AS revenue_7d_ago,
    revenue - LAG(revenue, 7, 0) OVER (ORDER BY sale_date ASC) AS wow_delta
FROM daily_sales
ORDER BY sale_date;
```

For this to give meaningful results, the table must have exactly one row per day with no gaps. If there are missing dates, consider filling gaps first or filtering carefully.

## Per-Partition LAG: Month-over-Month Per Product

Use `PARTITION BY` to compute independent lag sequences for each group. This example calculates month-over-month sales change for each product:

```sql
SELECT
    product_id,
    month,
    monthly_sales,
    LAG(monthly_sales) OVER (
        PARTITION BY product_id
        ORDER BY month ASC
    ) AS prev_month_sales,
    monthly_sales - LAG(monthly_sales) OVER (
        PARTITION BY product_id
        ORDER BY month ASC
    ) AS mom_delta
FROM product_monthly_sales
ORDER BY product_id, month;
```

`LAG()` resets at each `product_id` boundary - the first month for product B does not look back to the last month of product A.

## Detecting State Changes

`LAG()` excels at detecting transitions between states. This query identifies rows where the order status changed:

```sql
SELECT
    order_id,
    status_time,
    status,
    LAG(status) OVER (
        PARTITION BY order_id
        ORDER BY status_time ASC
    ) AS prev_status,
    CASE
        WHEN LAG(status) OVER (
            PARTITION BY order_id
            ORDER BY status_time ASC
        ) IS NULL THEN 'initial'
        WHEN status != LAG(status) OVER (
            PARTITION BY order_id
            ORDER BY status_time ASC
        ) THEN 'changed'
        ELSE 'unchanged'
    END AS transition_type
FROM order_status_history
ORDER BY order_id, status_time;
```

## Computing Time Between Events

Use `LAG()` on timestamp columns to measure the time elapsed between consecutive events per user:

```sql
SELECT
    user_id,
    event_time,
    event_type,
    LAG(event_time) OVER (
        PARTITION BY user_id
        ORDER BY event_time ASC
    ) AS prev_event_time,
    dateDiff(
        'second',
        LAG(event_time) OVER (
            PARTITION BY user_id
            ORDER BY event_time ASC
        ),
        event_time
    ) AS seconds_since_prev_event
FROM user_events
WHERE event_date = today() - 1
ORDER BY user_id, event_time;
```

Rows where `seconds_since_prev_event` exceeds a threshold (for example, 1800 seconds / 30 minutes) indicate the start of a new session - a common session segmentation technique.

## Comparing Multiple Lag Offsets Simultaneously

You can include multiple `LAG()` calls in the same query to compare against several historical periods at once:

```sql
SELECT
    metric_date,
    value,
    LAG(value, 1)  OVER (ORDER BY metric_date) AS value_1d_ago,
    LAG(value, 7)  OVER (ORDER BY metric_date) AS value_7d_ago,
    LAG(value, 30) OVER (ORDER BY metric_date) AS value_30d_ago,
    value - LAG(value, 1)  OVER (ORDER BY metric_date) AS delta_1d,
    value - LAG(value, 7)  OVER (ORDER BY metric_date) AS delta_7d,
    value - LAG(value, 30) OVER (ORDER BY metric_date) AS delta_30d
FROM daily_metrics
WHERE metric_name = 'active_users'
ORDER BY metric_date;
```

## Summary

`LAG()` is one of the most frequently used window functions in analytical SQL because it eliminates self-joins for any "current vs. previous" comparison. The three-argument form - `LAG(expr, offset, default)` - gives full control over how far back to look and what to show at partition boundaries. Combine it with `PARTITION BY` to compute independent lag sequences across groups, and with `dateDiff()` or arithmetic subtraction to produce deltas, percentage changes, and state transitions across time series data in ClickHouse.
