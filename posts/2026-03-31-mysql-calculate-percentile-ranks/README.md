# How to Calculate Percentile Ranks in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, Percentile, PERCENT_RANK, Analytics

Description: Learn how to calculate percentile ranks in MySQL using PERCENT_RANK(), NTILE(), and CUME_DIST() window functions for customer scoring and data distribution analysis.

---

## What Is a Percentile Rank?

A percentile rank tells you what percentage of values in a dataset fall below a given value. MySQL 8.0 provides `PERCENT_RANK()`, `CUME_DIST()`, and `NTILE()` to calculate these distributions without application-layer processing.

## PERCENT_RANK()

`PERCENT_RANK()` returns a value between 0 and 1 indicating where a row falls relative to all other rows:

```sql
SELECT
  customer_id,
  total_spent,
  ROUND(PERCENT_RANK() OVER (ORDER BY total_spent), 4) AS percentile_rank,
  ROUND(PERCENT_RANK() OVER (ORDER BY total_spent) * 100, 1) AS percentile_pct
FROM customer_totals
ORDER BY total_spent DESC;
```

Formula: `(rank - 1) / (total_rows - 1)`. The lowest value gets 0, the highest gets 1.

## NTILE() for Percentile Buckets

`NTILE(100)` divides rows into 100 equal buckets - a direct percentile assignment:

```sql
SELECT
  customer_id,
  total_spent,
  NTILE(100) OVER (ORDER BY total_spent) AS percentile_bucket,
  NTILE(4)   OVER (ORDER BY total_spent) AS quartile,
  NTILE(10)  OVER (ORDER BY total_spent) AS decile
FROM customer_totals
ORDER BY total_spent DESC;
```

Use `NTILE(4)` for quartiles (Q1-Q4), `NTILE(10)` for deciles.

## Finding the Nth Percentile Value

Calculate the value at the 90th, 95th, and 99th percentile:

```sql
WITH ranked AS (
  SELECT
    total_spent,
    PERCENT_RANK() OVER (ORDER BY total_spent) AS pct_rank
  FROM customer_totals
)
SELECT
  MAX(CASE WHEN pct_rank <= 0.90 THEN total_spent END) AS p90,
  MAX(CASE WHEN pct_rank <= 0.95 THEN total_spent END) AS p95,
  MAX(CASE WHEN pct_rank <= 0.99 THEN total_spent END) AS p99
FROM ranked;
```

## Percentile Rank Within Groups

Calculate percentile rank within each product category:

```sql
SELECT
  p.category,
  p.name,
  p.price,
  ROUND(PERCENT_RANK() OVER (
    PARTITION BY p.category
    ORDER BY p.price
  ) * 100, 1) AS price_percentile_within_category
FROM products p
ORDER BY p.category, p.price;
```

## Labeling Customers by Tier

Use `NTILE` to assign customer segments:

```sql
SELECT
  customer_id,
  total_spent,
  CASE NTILE(4) OVER (ORDER BY total_spent)
    WHEN 4 THEN 'Top 25%'
    WHEN 3 THEN 'Upper Middle'
    WHEN 2 THEN 'Lower Middle'
    WHEN 1 THEN 'Bottom 25%'
  END AS spending_tier
FROM customer_totals
ORDER BY total_spent DESC;
```

## Percentile Rank for Response Time Analysis

Apply to performance data to find slow outliers:

```sql
SELECT
  endpoint,
  response_time_ms,
  ROUND(PERCENT_RANK() OVER (ORDER BY response_time_ms) * 100, 1) AS latency_percentile
FROM api_request_log
ORDER BY response_time_ms DESC
LIMIT 100;
```

## Summary

Calculate percentile ranks in MySQL 8.0 using `PERCENT_RANK()` for relative position (0 to 1), `NTILE(N)` for bucket-based assignment (quartiles, deciles, percentiles), and `CUME_DIST()` for cumulative distribution. Use `PARTITION BY` to compute ranks independently per group, and `CASE WHEN` with `NTILE` to map buckets to business labels like "Top 25%".
