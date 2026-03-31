# How to Use NTILE() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, NTILE, Percentile Ranking, Analytics

Description: Learn how to use the NTILE() window function in ClickHouse to divide rows into equal-sized buckets for percentile ranking and cohort analysis.

---

## What Is NTILE()

`NTILE(n)` is a window function that divides rows within a partition into `n` roughly equal-sized buckets and assigns each row a bucket number from 1 to n. It is useful for:

- Creating percentile groups (quartiles, deciles, percentiles)
- Cohort analysis
- Distributing workloads evenly
- Scoring and ranking distributions

```sql
NTILE(n) OVER (
    [PARTITION BY partition_column]
    ORDER BY sort_column [ASC|DESC]
)
```

## Basic NTILE() Example

```sql
CREATE TABLE customer_orders (
    customer_id UInt64,
    total_spent Float64,
    order_count UInt32,
    country LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY customer_id;

-- Divide customers into 4 quartiles by spending
SELECT
    customer_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent ASC) AS spending_quartile
FROM customer_orders;
-- 1 = lowest spending quartile, 4 = highest
```

## Creating Percentiles with NTILE(100)

```sql
-- Assign each customer to a percentile (1-100)
SELECT
    customer_id,
    total_spent,
    NTILE(100) OVER (ORDER BY total_spent ASC) AS spending_percentile
FROM customer_orders;

-- Find customers in the top 10%
SELECT customer_id, total_spent
FROM (
    SELECT
        customer_id,
        total_spent,
        NTILE(100) OVER (ORDER BY total_spent ASC) AS pct
    FROM customer_orders
)
WHERE pct >= 90
ORDER BY total_spent DESC;
```

## NTILE() with PARTITION BY

```sql
-- Assign deciles within each country
SELECT
    customer_id,
    country,
    total_spent,
    NTILE(10) OVER (PARTITION BY country ORDER BY total_spent ASC) AS country_decile
FROM customer_orders;

-- Compare cross-country performance: find customers in top decile of their country
SELECT
    customer_id,
    country,
    total_spent
FROM (
    SELECT
        customer_id,
        country,
        total_spent,
        NTILE(10) OVER (PARTITION BY country ORDER BY total_spent ASC) AS decile
    FROM customer_orders
)
WHERE decile = 10  -- top 10% within their country
ORDER BY country, total_spent DESC;
```

## Labeling NTILE Buckets

```sql
-- Convert numeric tile to descriptive labels
SELECT
    customer_id,
    total_spent,
    tile,
    CASE tile
        WHEN 1 THEN 'Bronze'
        WHEN 2 THEN 'Silver'
        WHEN 3 THEN 'Gold'
        WHEN 4 THEN 'Platinum'
    END AS tier
FROM (
    SELECT
        customer_id,
        total_spent,
        NTILE(4) OVER (ORDER BY total_spent ASC) AS tile
    FROM customer_orders
)
ORDER BY total_spent DESC;
```

## Analyzing Distribution with NTILE

```sql
-- Understand revenue distribution across quartiles
SELECT
    quartile,
    count() AS customers,
    round(min(total_spent), 2) AS min_spend,
    round(max(total_spent), 2) AS max_spend,
    round(avg(total_spent), 2) AS avg_spend,
    round(sum(total_spent), 2) AS total_revenue,
    round(sum(total_spent) / total_all * 100, 1) AS pct_of_revenue
FROM (
    SELECT
        total_spent,
        NTILE(4) OVER (ORDER BY total_spent ASC) AS quartile
    FROM customer_orders
) t
CROSS JOIN (SELECT sum(total_spent) AS total_all FROM customer_orders)
GROUP BY quartile, total_all
ORDER BY quartile;
```

## Practical Example: User Engagement Scoring

```sql
CREATE TABLE user_metrics (
    user_id UInt64,
    sessions_30d UInt32,
    events_30d UInt32,
    days_active_30d UInt8,
    revenue_30d Float64
) ENGINE = MergeTree()
ORDER BY user_id;

-- Create engagement tiers
SELECT
    user_id,
    sessions_30d,
    events_30d,
    days_active_30d,
    -- Score based on multiple metrics
    NTILE(5) OVER (ORDER BY (
        sessions_30d + events_30d / 10 + days_active_30d * 2
    ) ASC) AS engagement_tier
FROM user_metrics;

-- Aggregate by engagement tier
SELECT
    engagement_tier,
    count() AS users,
    round(avg(revenue_30d), 2) AS avg_revenue,
    round(avg(sessions_30d), 1) AS avg_sessions,
    round(avg(days_active_30d), 1) AS avg_active_days
FROM (
    SELECT
        user_id,
        revenue_30d,
        sessions_30d,
        days_active_30d,
        NTILE(5) OVER (ORDER BY (
            sessions_30d + events_30d / 10 + days_active_30d * 2
        ) ASC) AS engagement_tier
    FROM user_metrics
)
GROUP BY engagement_tier
ORDER BY engagement_tier;
```

## How Uneven Bucket Sizes Work

When rows cannot be divided evenly, ClickHouse distributes the remainder to the first buckets:

```sql
-- With 10 rows and NTILE(3):
-- Bucket 1: rows 1-4 (4 rows)
-- Bucket 2: rows 5-7 (3 rows)
-- Bucket 3: rows 8-10 (3 rows)
-- (10 / 3 = 3 remainder 1, so first bucket gets +1)
SELECT
    val,
    NTILE(3) OVER (ORDER BY val) AS bucket
FROM (SELECT arrayJoin([1,2,3,4,5,6,7,8,9,10]) AS val);
```

## Summary

`NTILE(n)` in ClickHouse divides ordered rows into `n` equal-sized buckets and assigns each row a bucket number from 1 to n. Use it to create quartiles (`NTILE(4)`), deciles (`NTILE(10)`), or percentiles (`NTILE(100)`). Combine with `PARTITION BY` to create within-group distributions, and use `CASE` expressions to add descriptive tier labels. NTILE is especially useful for revenue segmentation, engagement scoring, and creating balanced cohorts for analysis.
