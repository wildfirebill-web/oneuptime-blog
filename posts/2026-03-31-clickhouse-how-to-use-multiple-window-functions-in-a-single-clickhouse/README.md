# How to Use Multiple Window Functions in a Single ClickHouse Query

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Functions, Query Optimization, Analytics, SQL

Description: Learn how to efficiently combine multiple window functions in a single ClickHouse query using the WINDOW clause, named windows, and performance best practices.

---

## Why Use Multiple Window Functions Together

Modern analytics often requires several different window-based calculations simultaneously - running totals, moving averages, rankings, and comparisons. Running these as separate queries is inefficient. ClickHouse can compute multiple window functions in a single pass.

## Basic Multi-Window Function Query

```sql
CREATE TABLE daily_revenue (
    sale_date Date,
    product_id UInt32,
    category LowCardinality(String),
    revenue Float64,
    orders UInt32
) ENGINE = MergeTree()
ORDER BY (sale_date, product_id);

-- Multiple window functions in one SELECT
SELECT
    sale_date,
    product_id,
    revenue,
    -- Ranking
    ROW_NUMBER() OVER (ORDER BY revenue DESC) AS overall_rank,
    RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS category_rank,
    -- Running aggregates
    sum(revenue) OVER (ORDER BY sale_date) AS cumulative_revenue,
    avg(revenue) OVER (ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS sma_7d,
    -- Comparison with previous period
    LAG(revenue, 7) OVER (PARTITION BY product_id ORDER BY sale_date) AS revenue_7d_ago,
    -- Percentile ranking
    NTILE(4) OVER (ORDER BY revenue) AS revenue_quartile
FROM daily_revenue
ORDER BY sale_date, product_id;
```

## Using Named Windows (WINDOW Clause)

When multiple functions share the same window definition, use the `WINDOW` clause to avoid repetition:

```sql
SELECT
    sale_date,
    revenue,
    -- All reference named windows
    sum(revenue) OVER w_date AS running_total,
    avg(revenue) OVER w_date AS running_avg,
    max(revenue) OVER w_date AS running_max,
    ROW_NUMBER() OVER w_date AS seq_num,
    -- Rolling windows
    avg(revenue) OVER w_7d AS sma_7d,
    sum(revenue) OVER w_7d AS sum_7d
FROM daily_revenue
WINDOW
    w_date AS (ORDER BY sale_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW),
    w_7d AS (ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
ORDER BY sale_date;
```

## Different Partitions in Same Query

```sql
-- Mix global and partitioned windows
SELECT
    sale_date,
    product_id,
    category,
    revenue,
    -- Global rank across all products
    RANK() OVER (ORDER BY revenue DESC) AS global_rank,
    -- Rank within category
    RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS category_rank,
    -- Running total for the product
    sum(revenue) OVER (PARTITION BY product_id ORDER BY sale_date) AS product_cumulative,
    -- Running total for the category
    sum(revenue) OVER (PARTITION BY category ORDER BY sale_date) AS category_cumulative,
    -- Running total across all products
    sum(revenue) OVER (ORDER BY sale_date, product_id) AS global_cumulative,
    -- Comparison within product
    LAG(revenue, 1) OVER (PARTITION BY product_id ORDER BY sale_date) AS prev_day_revenue
FROM daily_revenue
ORDER BY sale_date, product_id;
```

## Performance: Window Sorting Cost

```sql
-- Each distinct window ordering requires a sort
-- Group similar windows to minimize sorts

-- INEFFICIENT: 4 different sort orders
SELECT
    ROW_NUMBER() OVER (ORDER BY revenue DESC) AS rank_by_rev,
    ROW_NUMBER() OVER (ORDER BY orders DESC) AS rank_by_orders,
    sum(revenue) OVER (PARTITION BY category ORDER BY sale_date) AS cat_running,
    avg(revenue) OVER (PARTITION BY product_id ORDER BY sale_date) AS prod_running_avg
FROM daily_revenue;

-- BETTER: separate queries or pre-sort in subquery
-- when window sets share ordering, they are computed together
SELECT
    sale_date,
    product_id,
    category,
    revenue,
    orders,
    -- These share the same window, computed in one sort
    sum(revenue) OVER w AS cum_rev,
    avg(revenue) OVER w AS avg_rev,
    max(revenue) OVER w AS max_rev,
    ROW_NUMBER() OVER w AS row_num
FROM daily_revenue
WINDOW w AS (PARTITION BY category ORDER BY sale_date)
ORDER BY category, sale_date;
```

## Practical Example: Complete Business Dashboard Query

```sql
SELECT
    sale_date,
    product_id,
    category,
    revenue,
    orders,
    -- Day-over-day comparison
    LAG(revenue, 1, 0) OVER w_product AS prev_day_rev,
    round(revenue - LAG(revenue, 1, 0) OVER w_product, 2) AS daily_delta,
    -- 7-day trends
    round(avg(revenue) OVER w_product_7d, 2) AS revenue_sma_7d,
    round(avg(orders) OVER w_product_7d, 1) AS orders_sma_7d,
    -- Cumulative year-to-date
    sum(revenue) OVER w_product_ytd AS ytd_revenue,
    sum(orders) OVER w_product_ytd AS ytd_orders,
    -- Rankings
    RANK() OVER w_cat_rank AS category_rank,
    NTILE(5) OVER (ORDER BY revenue) AS revenue_quintile,
    -- Share of category
    round(revenue / sum(revenue) OVER w_cat_total * 100, 1) AS pct_of_category
FROM daily_revenue
WINDOW
    w_product AS (PARTITION BY product_id ORDER BY sale_date),
    w_product_7d AS (PARTITION BY product_id ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW),
    w_product_ytd AS (PARTITION BY product_id, toYear(sale_date) ORDER BY sale_date),
    w_cat_rank AS (PARTITION BY category, sale_date ORDER BY revenue DESC),
    w_cat_total AS (PARTITION BY category, sale_date)
ORDER BY sale_date, category, product_id;
```

## Summary

ClickHouse supports multiple window functions in a single query, enabling rich analytics without multiple passes. The `WINDOW` clause lets you define named window specifications that multiple functions can reference, improving readability and allowing ClickHouse to optimize shared sorts. Mix partitioned and global windows, different frame sizes, and different functions (ranking, running aggregates, LAG/LEAD) in a single SELECT. Group functions that share the same window definition to minimize the number of sorts ClickHouse must perform.
