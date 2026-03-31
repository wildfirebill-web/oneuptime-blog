# How to Calculate Month-over-Month Growth Rate in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Growth Rate, Analytics, Window Function, SQL

Description: Calculate month-over-month growth rates in ClickHouse using window functions and LAG to compare current period metrics against the previous period.

---

## Month-over-Month Formula

MoM Growth = (Current Month Value - Previous Month Value) / Previous Month Value * 100

## Calculating MoM Revenue Growth

```sql
WITH monthly AS (
    SELECT
        toStartOfMonth(created_at)  AS month,
        sum(amount)                 AS revenue
    FROM transactions
    WHERE created_at >= today() - INTERVAL 13 MONTH
    GROUP BY month
    ORDER BY month
),
with_prev AS (
    SELECT
        month,
        revenue,
        lagInFrame(revenue) OVER (ORDER BY month ROWS BETWEEN 1 PRECEDING AND CURRENT ROW)
            AS prev_revenue
    FROM monthly
)
SELECT
    month,
    round(revenue, 2)           AS revenue,
    round(prev_revenue, 2)      AS prev_revenue,
    round(
        (revenue - prev_revenue) / prev_revenue * 100,
        2
    )                           AS mom_growth_pct
FROM with_prev
WHERE prev_revenue IS NOT NULL
ORDER BY month;
```

## MoM User Growth

```sql
WITH monthly_users AS (
    SELECT
        toStartOfMonth(created_at) AS month,
        uniq(user_id)              AS users
    FROM events
    WHERE created_at >= today() - INTERVAL 13 MONTH
    GROUP BY month
)
SELECT
    month,
    users,
    lagInFrame(users) OVER (ORDER BY month) AS prev_users,
    round((users - lagInFrame(users) OVER (ORDER BY month))
          * 100.0
          / lagInFrame(users) OVER (ORDER BY month), 2)  AS mom_pct
FROM monthly_users
ORDER BY month;
```

## Compound Average Monthly Growth Rate

If you want the average MoM growth over N months:

```sql
WITH monthly AS (
    SELECT
        toStartOfMonth(created_at) AS month,
        sum(amount)                AS revenue
    FROM transactions
    GROUP BY month
    ORDER BY month
),
bounds AS (
    SELECT
        min(revenue)    AS first_val,
        max(revenue)    AS last_val,  -- assumes last month is the latest
        count() - 1     AS periods
    FROM monthly
    WHERE month >= today() - INTERVAL 12 MONTH
)
SELECT
    round(pow(last_val / first_val, 1.0 / periods) - 1, 4) * 100 AS avg_monthly_growth_pct
FROM bounds;
```

## Week-over-Week Variant

Replace `toStartOfMonth` with `toStartOfWeek` and adjust the lookback window.

```sql
WITH weekly AS (
    SELECT toStartOfWeek(ts) AS week, count() AS events
    FROM page_views
    WHERE ts >= today() - 14
    GROUP BY week
)
SELECT
    week,
    events,
    lagInFrame(events) OVER (ORDER BY week) AS prev_week,
    round((events - lagInFrame(events) OVER (ORDER BY week)) * 100.0
          / lagInFrame(events) OVER (ORDER BY week), 2) AS wow_pct
FROM weekly
ORDER BY week;
```

## Summary

ClickHouse window functions like `lagInFrame` make MoM growth calculations straightforward. Use CTEs to first aggregate into time buckets, then apply LAG in a second pass to compare adjacent periods. This pattern works for revenue, users, events, or any time-series metric.
