# How to Build Comparison Reports (Period over Period) in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Period Comparison, Year over Year, Week over Week, Analytics

Description: Learn how to build period-over-period comparison reports in ClickHouse to compare metrics across weeks, months, or years using self-joins and window functions.

---

## Period-over-Period Analysis

Period-over-period (PoP) comparisons reveal growth trends - this week vs last week, this month vs the same month last year. ClickHouse makes these comparisons efficient via conditional aggregation and self-joins on date-shifted keys.

## Sample Revenue Table

```sql
CREATE TABLE daily_revenue
(
    day Date,
    region String,
    revenue Float64
)
ENGINE = MergeTree()
ORDER BY (day, region);
```

## Week-over-Week via Self-Join

```sql
SELECT
    current.day,
    current.region,
    current.revenue AS this_week,
    prior.revenue AS last_week,
    round((current.revenue - prior.revenue) / prior.revenue * 100, 2) AS pct_change
FROM daily_revenue current
LEFT JOIN daily_revenue prior
    ON current.region = prior.region
    AND current.day = prior.day + INTERVAL 7 DAY
WHERE current.day >= today() - 7
ORDER BY current.day, current.region;
```

## Month-over-Month with Conditional Aggregation

```sql
SELECT
    region,
    sumIf(revenue, toStartOfMonth(day) = toStartOfMonth(today())) AS this_month,
    sumIf(revenue, toStartOfMonth(day) = toStartOfMonth(today() - INTERVAL 1 MONTH)) AS last_month,
    round(
        (sumIf(revenue, toStartOfMonth(day) = toStartOfMonth(today())) -
         sumIf(revenue, toStartOfMonth(day) = toStartOfMonth(today() - INTERVAL 1 MONTH))) /
         sumIf(revenue, toStartOfMonth(day) = toStartOfMonth(today() - INTERVAL 1 MONTH)) * 100, 2
    ) AS mom_pct
FROM daily_revenue
WHERE day >= toStartOfMonth(today() - INTERVAL 1 MONTH)
GROUP BY region
ORDER BY mom_pct DESC;
```

## Year-over-Year Comparison

```sql
SELECT
    toStartOfMonth(day) AS month,
    sumIf(revenue, toYear(day) = toYear(today())) AS this_year,
    sumIf(revenue, toYear(day) = toYear(today()) - 1) AS last_year,
    round(
        (sumIf(revenue, toYear(day) = toYear(today())) -
         sumIf(revenue, toYear(day) = toYear(today()) - 1)) /
         nullIf(sumIf(revenue, toYear(day) = toYear(today()) - 1), 0) * 100, 2
    ) AS yoy_pct
FROM daily_revenue
WHERE toYear(day) IN (toYear(today()), toYear(today()) - 1)
GROUP BY month
ORDER BY month;
```

## Using LAG for Rolling Comparison

```sql
SELECT
    day,
    sum(revenue) AS revenue,
    lagInFrame(sum(revenue), 7) OVER (ORDER BY day) AS revenue_7d_ago,
    round(
        (sum(revenue) - lagInFrame(sum(revenue), 7) OVER (ORDER BY day)) /
        lagInFrame(sum(revenue), 7) OVER (ORDER BY day) * 100, 2
    ) AS wow_pct
FROM daily_revenue
GROUP BY day
ORDER BY day;
```

## Showing Both Periods Side by Side

Use UNION ALL to stack periods, then pivot in your BI tool:

```sql
SELECT 'This Month' AS period, sum(revenue) AS revenue FROM daily_revenue
WHERE toStartOfMonth(day) = toStartOfMonth(today())
UNION ALL
SELECT 'Last Month', sum(revenue) FROM daily_revenue
WHERE toStartOfMonth(day) = toStartOfMonth(today() - INTERVAL 1 MONTH);
```

## Summary

Period-over-period reports in ClickHouse use self-joins with date arithmetic, conditional aggregation with `sumIf`, or window functions with `lagInFrame`. All three approaches handle week-over-week, month-over-month, and year-over-year comparisons efficiently.
