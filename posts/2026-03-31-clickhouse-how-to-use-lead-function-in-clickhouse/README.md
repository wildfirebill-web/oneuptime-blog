# How to Use LEAD() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Functions, LEAD, Time Series, Analytics

Description: Learn how to use the LEAD() window function in ClickHouse to access values from subsequent rows for forecasting, lookahead, and session duration calculations.

---

## What Is LEAD()

`LEAD(expr, offset, default)` is a window function that returns the value of `expr` from a row that is `offset` rows **after** the current row. It is the forward-looking counterpart to `LAG()`.

```sql
LEAD(expr [, offset [, default]]) OVER (
    [PARTITION BY partition_column]
    ORDER BY sort_column
)
```

- `expr`: the column or expression to look ahead at
- `offset`: how many rows to look forward (default: 1)
- `default`: value returned for the last rows where no next row exists (default: NULL)

## Basic LEAD() Example

```sql
CREATE TABLE events (
    user_id UInt64,
    ts DateTime,
    event_type LowCardinality(String),
    page String
) ENGINE = MergeTree()
ORDER BY (user_id, ts);

-- See what page a user visits next
SELECT
    user_id,
    ts,
    page AS current_page,
    LEAD(page, 1) OVER (PARTITION BY user_id ORDER BY ts) AS next_page
FROM events
WHERE event_type = 'pageview'
ORDER BY user_id, ts;
```

## Calculating Session Duration

```sql
-- Session duration: time until the next event in the session
SELECT
    user_id,
    session_id,
    ts AS session_start,
    LEAD(ts, 1) OVER (PARTITION BY user_id ORDER BY ts) AS next_event_ts,
    dateDiff('second',
        ts,
        LEAD(ts, 1) OVER (PARTITION BY user_id ORDER BY ts)
    ) AS seconds_until_next_event
FROM user_events
ORDER BY user_id, ts;

-- Last event in each session: LEAD returns NULL (no next event)
SELECT user_id, ts AS session_end_ts
FROM (
    SELECT
        user_id,
        ts,
        LEAD(ts, 1) OVER (PARTITION BY user_id ORDER BY ts) AS next_ts
    FROM user_events
)
WHERE next_ts IS NULL;
```

## Page Flow Analysis with LEAD()

```sql
-- Which pages lead to conversions?
SELECT
    current_page,
    next_page,
    count() AS transitions,
    countIf(next_page = '/checkout') AS led_to_checkout,
    round(countIf(next_page = '/checkout') / count() * 100, 1) AS checkout_rate_pct
FROM (
    SELECT
        page AS current_page,
        LEAD(page, 1) OVER (PARTITION BY session_id ORDER BY ts) AS next_page
    FROM pageviews
)
WHERE next_page IS NOT NULL
GROUP BY current_page, next_page
ORDER BY led_to_checkout DESC
LIMIT 20;
```

## LEAD() with Multiple Offsets

```sql
-- Look ahead multiple steps
SELECT
    date,
    value,
    LEAD(value, 1) OVER (ORDER BY date) AS next_1d,
    LEAD(value, 7) OVER (ORDER BY date) AS next_7d,
    LEAD(value, 30) OVER (ORDER BY date) AS next_30d,
    -- Future change calculation
    round(LEAD(value, 7) OVER (ORDER BY date) - value, 2) AS change_in_7d
FROM time_series
ORDER BY date;
```

## Detecting Intervals with LEAD()

```sql
CREATE TABLE maintenance_windows (
    server_id UInt32,
    start_ts DateTime,
    end_ts DateTime
) ENGINE = MergeTree()
ORDER BY (server_id, start_ts);

-- Gap between consecutive maintenance windows
SELECT
    server_id,
    end_ts,
    LEAD(start_ts, 1) OVER (PARTITION BY server_id ORDER BY start_ts) AS next_start,
    dateDiff('hour',
        end_ts,
        LEAD(start_ts, 1) OVER (PARTITION BY server_id ORDER BY start_ts)
    ) AS hours_until_next_maintenance
FROM maintenance_windows
WHERE LEAD(start_ts, 1) OVER (PARTITION BY server_id ORDER BY start_ts) IS NOT NULL
ORDER BY server_id, start_ts;
```

## Practical Example: Sales Forecasting Comparison

```sql
CREATE TABLE monthly_sales (
    month Date,
    product_id UInt32,
    revenue Float64,
    forecast Float64
) ENGINE = MergeTree()
ORDER BY (product_id, month);

-- Compare actual next month vs forecast for this month
SELECT
    month,
    product_id,
    revenue AS current_revenue,
    forecast AS this_month_forecast,
    LEAD(revenue, 1) OVER (PARTITION BY product_id ORDER BY month) AS actual_next_month,
    -- Accuracy: how close was this month's forecast to actual next month?
    round(abs(forecast - LEAD(revenue, 1) OVER (PARTITION BY product_id ORDER BY month)) /
        nullIf(LEAD(revenue, 1) OVER (PARTITION BY product_id ORDER BY month), 0) * 100,
        1) AS forecast_error_pct
FROM monthly_sales
WHERE LEAD(revenue, 1) OVER (PARTITION BY product_id ORDER BY month) IS NOT NULL
ORDER BY product_id, month;
```

## LEAD() vs LAG() Summary

| Function | Direction | Use Case |
|----------|-----------|---------|
| `LAG()` | Look back | Period-over-period comparison, change from past |
| `LEAD()` | Look forward | Next step prediction, duration until next event |

```sql
-- Often used together for delta calculations
SELECT
    date,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY date) AS yesterday,
    LEAD(revenue, 1, 0) OVER (ORDER BY date) AS tomorrow,
    revenue - LAG(revenue, 1, 0) OVER (ORDER BY date) AS change_from_yesterday,
    LEAD(revenue, 1, 0) OVER (ORDER BY date) - revenue AS expected_change_tomorrow
FROM daily_revenue
ORDER BY date;
```

## Summary

`LEAD()` in ClickHouse accesses values from subsequent rows within an ordered window, making it ideal for session duration calculations, page flow analysis, and future period comparisons. The default value parameter handles the last rows where no next value exists. Use `LEAD()` with `PARTITION BY` to analyze sequences within groups (sessions, users, products), and combine it with `LAG()` when you need both backward and forward context in the same query.
