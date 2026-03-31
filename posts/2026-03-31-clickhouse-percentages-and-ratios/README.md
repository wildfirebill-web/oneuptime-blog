# How to Calculate Percentages and Ratios in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Percentage, Ratio, Conditional Aggregation, Analytics

Description: Learn how to calculate percentages, share of total, and ratios in ClickHouse using window functions, conditional aggregation, and countIf patterns.

---

## Percentages and Ratios in Analytics

Percentages and ratios contextualize raw counts. Revenue share by channel, error rate by service, and conversion rate by cohort are all percentage calculations. ClickHouse provides efficient primitives for computing them.

## Share of Total

Calculate each category's share of total revenue:

```sql
SELECT
    category,
    sum(revenue) AS category_revenue,
    round(
        sum(revenue) / sum(sum(revenue)) OVER () * 100,
        2
    ) AS revenue_share_pct
FROM sales
WHERE event_time >= today() - 30
GROUP BY category
ORDER BY revenue_share_pct DESC;
```

## Conditional Count Ratio

Calculate conversion rate as a ratio of two event types:

```sql
SELECT
    toDate(event_time) AS day,
    countIf(event_type = 'checkout_complete') AS conversions,
    countIf(event_type = 'checkout_start') AS starts,
    round(
        countIf(event_type = 'checkout_complete') /
        nullIf(countIf(event_type = 'checkout_start'), 0) * 100,
        2
    ) AS conversion_rate_pct
FROM user_events
WHERE event_time >= today() - 30
GROUP BY day
ORDER BY day;
```

## Error Rate

Calculate error rate as a percentage of total requests:

```sql
SELECT
    service,
    toStartOfHour(event_time) AS hour,
    countIf(status_code >= 500) AS errors,
    count() AS total_requests,
    round(countIf(status_code >= 500) / count() * 100, 4) AS error_rate_pct
FROM http_logs
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY service, hour
ORDER BY service, hour;
```

## Percentage Change

Calculate month-over-month percentage change:

```sql
SELECT
    month,
    revenue,
    prev_revenue,
    round(
        (revenue - prev_revenue) / nullIf(prev_revenue, 0) * 100,
        2
    ) AS mom_pct_change
FROM (
    SELECT
        toYYYYMM(event_time) AS month,
        sum(revenue) AS revenue,
        lag(sum(revenue), 1) OVER (ORDER BY toYYYYMM(event_time)) AS prev_revenue
    FROM sales
    GROUP BY month
)
ORDER BY month;
```

## Row Percentage Within Group

Show each row as a percentage of its group total:

```sql
SELECT
    channel,
    country,
    users,
    round(users / sum(users) OVER (PARTITION BY channel) * 100, 2) AS pct_within_channel
FROM (
    SELECT channel, country, count() AS users
    FROM user_acquisition
    WHERE signup_time >= today() - 30
    GROUP BY channel, country
)
ORDER BY channel, users DESC;
```

## Ratio with Safe Division

Always use `nullIf` to avoid division by zero:

```sql
SELECT
    product_id,
    views,
    purchases,
    round(purchases / nullIf(views, 0) * 100, 3) AS purchase_rate_pct
FROM (
    SELECT
        product_id,
        countIf(event_type = 'view') AS views,
        countIf(event_type = 'purchase') AS purchases
    FROM product_events
    WHERE event_time >= today() - 7
    GROUP BY product_id
)
ORDER BY purchase_rate_pct DESC;
```

## Summary

ClickHouse calculates percentages using `sum() OVER ()` for share of total, `countIf` ratios for event-based rates, window function `lag()` for period-over-period change, and `nullIf` for safe division. Partitioned window sums enable row-within-group percentages. These patterns are the building blocks of virtually every analytics dashboard.
