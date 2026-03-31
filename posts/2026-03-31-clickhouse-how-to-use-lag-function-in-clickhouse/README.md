# How to Use LAG() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Functions, LAG, Time Series, Analytics

Description: Learn how to use the LAG() window function in ClickHouse to access values from previous rows for period-over-period comparisons and change detection.

---

## What Is LAG()

`LAG(expr, offset, default)` is a window function that returns the value of `expr` from a row that is `offset` rows before the current row in the window's ordering. It is essential for period-over-period comparisons and detecting changes.

```sql
LAG(expr [, offset [, default]]) OVER (
    [PARTITION BY partition_column]
    ORDER BY sort_column
)
```

- `expr`: the column or expression to look back at
- `offset`: how many rows to look back (default: 1)
- `default`: value to return if no previous row exists (default: NULL)

## Basic LAG() Example

```sql
CREATE TABLE daily_metrics (
    date Date,
    revenue Float64,
    users UInt32,
    events UInt64
) ENGINE = MergeTree()
ORDER BY date;

-- Compare today's revenue with yesterday's
SELECT
    date,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY date) AS prev_day_revenue,
    revenue - LAG(revenue, 1, 0) OVER (ORDER BY date) AS revenue_change
FROM daily_metrics
ORDER BY date;
```

## Period-over-Period Growth

```sql
-- Month-over-month growth rate
SELECT
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) AS prev_revenue,
    round(
        (revenue - LAG(revenue, 1) OVER (ORDER BY date)) /
        LAG(revenue, 1) OVER (ORDER BY date) * 100,
        2
    ) AS mom_growth_pct
FROM monthly_revenue
ORDER BY date;

-- Year-over-year comparison (12 periods back)
SELECT
    month,
    revenue,
    LAG(revenue, 12, 0) OVER (ORDER BY month) AS same_month_last_year,
    round(
        (revenue - LAG(revenue, 12, 0) OVER (ORDER BY month)) /
        nullIf(LAG(revenue, 12, 0) OVER (ORDER BY month), 0) * 100,
        1
    ) AS yoy_growth_pct
FROM monthly_revenue
ORDER BY month;
```

## LAG() with PARTITION BY

```sql
-- Compare each product's daily sales with the previous day (per product)
SELECT
    product_id,
    date,
    sales,
    LAG(sales, 1, 0) OVER (PARTITION BY product_id ORDER BY date) AS prev_sales,
    sales - LAG(sales, 1, 0) OVER (PARTITION BY product_id ORDER BY date) AS sales_delta
FROM product_daily_sales
ORDER BY product_id, date;
```

## Detecting State Changes with LAG()

```sql
CREATE TABLE server_status (
    server_id UInt32,
    ts DateTime,
    status LowCardinality(String)  -- 'up', 'down', 'degraded'
) ENGINE = MergeTree()
ORDER BY (server_id, ts);

-- Find when status changed
SELECT
    server_id,
    ts,
    status,
    LAG(status, 1) OVER (PARTITION BY server_id ORDER BY ts) AS prev_status,
    (status != LAG(status, 1) OVER (PARTITION BY server_id ORDER BY ts)) AS status_changed
FROM server_status;

-- Only show status transitions
SELECT server_id, ts, prev_status, status AS new_status
FROM (
    SELECT
        server_id,
        ts,
        status,
        LAG(status) OVER (PARTITION BY server_id ORDER BY ts) AS prev_status
    FROM server_status
)
WHERE prev_status IS NOT NULL AND status != prev_status
ORDER BY server_id, ts;
```

## LAG() with Multiple Offsets

```sql
-- Compare with multiple previous periods at once
SELECT
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) AS revenue_1d_ago,
    LAG(revenue, 7) OVER (ORDER BY date) AS revenue_7d_ago,
    LAG(revenue, 30) OVER (ORDER BY date) AS revenue_30d_ago,
    -- Change calculations
    round(revenue - LAG(revenue, 7) OVER (ORDER BY date), 2) AS wow_change,
    round(revenue - LAG(revenue, 30) OVER (ORDER BY date), 2) AS mom_change
FROM daily_revenue
ORDER BY date;
```

## Practical Example: User Retention Analysis

```sql
CREATE TABLE user_activity (
    user_id UInt64,
    activity_date Date,
    session_count UInt16
) ENGINE = MergeTree()
ORDER BY (user_id, activity_date);

-- Days since user's last activity
SELECT
    user_id,
    activity_date,
    LAG(activity_date, 1) OVER (PARTITION BY user_id ORDER BY activity_date) AS prev_activity,
    dateDiff('day',
        LAG(activity_date, 1) OVER (PARTITION BY user_id ORDER BY activity_date),
        activity_date
    ) AS days_since_last_activity
FROM user_activity;

-- Identify churned sessions: gap > 30 days
SELECT
    user_id,
    activity_date AS return_date,
    prev_activity,
    days_gap,
    'churned_user_returned' AS event_type
FROM (
    SELECT
        user_id,
        activity_date,
        LAG(activity_date, 1) OVER (PARTITION BY user_id ORDER BY activity_date) AS prev_activity,
        dateDiff('day',
            LAG(activity_date, 1) OVER (PARTITION BY user_id ORDER BY activity_date),
            activity_date
        ) AS days_gap
    FROM user_activity
)
WHERE days_gap > 30
ORDER BY return_date;
```

## Summary

`LAG()` in ClickHouse retrieves values from previous rows in an ordered window, making it the primary tool for period-over-period comparisons, state change detection, and gap analysis. Use the offset parameter to look back multiple periods (e.g., `LAG(revenue, 12)` for year-over-year). Combine with `PARTITION BY` to apply comparisons within groups like products or users. Provide a default value as the third argument to handle the first row where no previous value exists.
