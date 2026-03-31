# How to Calculate Rate of Change in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Rate of Change, Window Function, Time-Series, Analytics

Description: Learn how to calculate rate of change, period-over-period growth, and derivatives in ClickHouse using window functions and lag expressions.

---

## Rate of Change in Analytics

Rate of change measures how quickly a metric grows or shrinks between time periods. It is fundamental for growth analysis, anomaly detection, and trend identification. ClickHouse window functions make this straightforward.

## Simple Period-over-Period Change

Calculate the absolute change from the previous day:

```sql
SELECT
    day,
    revenue,
    revenue - lag(revenue, 1) OVER (ORDER BY day) AS daily_change,
    revenue - lag(revenue, 1) OVER (ORDER BY day) AS abs_delta
FROM (
    SELECT toDate(event_time) AS day, sum(revenue) AS revenue
    FROM sales
    WHERE event_time >= today() - 30
    GROUP BY day
)
ORDER BY day;
```

## Percentage Rate of Change

Calculate percentage growth relative to the previous period:

```sql
SELECT
    day,
    revenue,
    prev_revenue,
    round((revenue - prev_revenue) / prev_revenue * 100, 2) AS pct_change
FROM (
    SELECT
        day,
        revenue,
        lag(revenue, 1, revenue) OVER (ORDER BY day) AS prev_revenue
    FROM (
        SELECT toDate(event_time) AS day, sum(revenue) AS revenue
        FROM sales GROUP BY day ORDER BY day
    )
)
WHERE prev_revenue > 0
ORDER BY day;
```

## Week-over-Week Growth

Compare to the same day last week:

```sql
SELECT
    day,
    revenue,
    lag(revenue, 7) OVER (ORDER BY day) AS wow_baseline,
    round(
        (revenue - lag(revenue, 7) OVER (ORDER BY day)) /
        lag(revenue, 7) OVER (ORDER BY day) * 100, 2
    ) AS wow_pct
FROM (
    SELECT toDate(event_time) AS day, sum(revenue) AS revenue
    FROM sales
    WHERE event_time >= today() - 60
    GROUP BY day
)
ORDER BY day;
```

## Year-over-Year Growth

Compare current month to the same month last year:

```sql
SELECT
    toYYYYMM(event_time) AS month,
    sum(revenue) AS revenue,
    lagInFrame(sum(revenue), 12) OVER (ORDER BY toYYYYMM(event_time)) AS yoy_baseline,
    round(
        (sum(revenue) - lagInFrame(sum(revenue), 12) OVER (ORDER BY toYYYYMM(event_time)))
        / lagInFrame(sum(revenue), 12) OVER (ORDER BY toYYYYMM(event_time)) * 100, 2
    ) AS yoy_pct
FROM sales
GROUP BY month
ORDER BY month;
```

## Acceleration (Second Derivative)

Find metrics that are accelerating or decelerating:

```sql
SELECT
    day,
    revenue,
    revenue - lag(revenue, 1) OVER (ORDER BY day) AS velocity,
    (revenue - lag(revenue, 1) OVER (ORDER BY day)) -
    lag(revenue - lag(revenue, 1) OVER (ORDER BY day), 1) OVER (ORDER BY day) AS acceleration
FROM (
    SELECT toDate(event_time) AS day, sum(revenue) AS revenue
    FROM sales GROUP BY day
)
ORDER BY day;
```

## Summary

ClickHouse calculates rate of change using `lag()` and `lagInFrame()` window functions, enabling daily, weekly, and year-over-year comparisons. Second-derivative calculations identify acceleration in growth trends. These patterns are the foundation of growth analytics dashboards.
