# How to Use MySQL for Reporting Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Reporting, Window Function, Aggregation

Description: Learn how to write effective MySQL reporting queries using GROUP BY, window functions, CTEs, and conditional aggregation to generate business reports.

---

## Foundations of MySQL Reporting

MySQL 8.0 introduced window functions, CTEs, and other analytical SQL features that make reporting queries much more expressive than older versions. Combining these with standard `GROUP BY` aggregations covers the vast majority of business reporting needs.

## Basic Aggregation Report

```sql
-- Monthly revenue and order count by region
SELECT
  DATE_FORMAT(order_date, '%Y-%m')  AS month,
  region,
  COUNT(*)                           AS order_count,
  SUM(revenue)                       AS total_revenue,
  AVG(revenue)                       AS avg_order_value,
  MIN(revenue)                       AS min_order,
  MAX(revenue)                       AS max_order
FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31'
  AND status = 'COMPLETED'
GROUP BY month, region
ORDER BY month, total_revenue DESC;
```

## Running Totals with Window Functions

```sql
-- Cumulative revenue by month
SELECT
  DATE_FORMAT(order_date, '%Y-%m')           AS month,
  SUM(revenue)                               AS monthly_revenue,
  SUM(SUM(revenue)) OVER (
    ORDER BY DATE_FORMAT(order_date, '%Y-%m')
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  )                                          AS cumulative_revenue
FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31'
  AND status = 'COMPLETED'
GROUP BY month
ORDER BY month;
```

## Pivot Report with Conditional Aggregation

```sql
-- Revenue by quarter, pivoted by region
SELECT
  YEAR(order_date)                                AS year,
  QUARTER(order_date)                             AS quarter,
  SUM(CASE WHEN region = 'EMEA'  THEN revenue ELSE 0 END) AS emea_revenue,
  SUM(CASE WHEN region = 'AMER'  THEN revenue ELSE 0 END) AS amer_revenue,
  SUM(CASE WHEN region = 'APAC'  THEN revenue ELSE 0 END) AS apac_revenue,
  SUM(revenue)                                    AS total_revenue
FROM orders
WHERE status = 'COMPLETED'
GROUP BY year, quarter
ORDER BY year, quarter;
```

## Year-Over-Year Comparison

```sql
-- YoY growth using LAG window function
WITH monthly AS (
  SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    YEAR(order_date)                 AS yr,
    MONTH(order_date)                AS mo,
    SUM(revenue)                     AS revenue
  FROM orders
  WHERE status = 'COMPLETED'
  GROUP BY month, yr, mo
)
SELECT
  month,
  revenue                                           AS current_revenue,
  LAG(revenue, 12) OVER (ORDER BY month)           AS prior_year_revenue,
  ROUND(
    (revenue - LAG(revenue, 12) OVER (ORDER BY month))
    / NULLIF(LAG(revenue, 12) OVER (ORDER BY month), 0) * 100,
  2)                                               AS yoy_pct_change
FROM monthly
ORDER BY month;
```

## Top-N Per Group

```sql
-- Top 3 products by revenue per region in 2025
WITH ranked AS (
  SELECT
    region,
    product_id,
    SUM(revenue)   AS revenue,
    RANK() OVER (
      PARTITION BY region
      ORDER BY SUM(revenue) DESC
    )              AS rnk
  FROM orders
  WHERE YEAR(order_date) = 2025
    AND status = 'COMPLETED'
  GROUP BY region, product_id
)
SELECT region, product_id, revenue, rnk
FROM ranked
WHERE rnk <= 3
ORDER BY region, rnk;
```

## Percent of Total

```sql
-- Each product's share of total revenue
SELECT
  product_id,
  SUM(revenue)                             AS product_revenue,
  ROUND(
    SUM(revenue) / SUM(SUM(revenue)) OVER () * 100,
  2)                                       AS pct_of_total
FROM orders
WHERE YEAR(order_date) = 2025
  AND status = 'COMPLETED'
GROUP BY product_id
ORDER BY product_revenue DESC;
```

## Performance Tips

```sql
-- Cover the filter and grouping columns in one index
ALTER TABLE orders
  ADD INDEX idx_report (order_date, status, region, revenue);

-- Use EXPLAIN to verify index usage
EXPLAIN SELECT DATE_FORMAT(order_date, '%Y-%m'), region, SUM(revenue)
FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31'
  AND status = 'COMPLETED'
GROUP BY 1, 2;
```

## Summary

MySQL 8.0 provides all the tools needed for business reporting: GROUP BY for basic aggregations, window functions for running totals and rankings, CTEs for readable multi-step queries, and conditional aggregation for pivot tables. Build covering indexes on filter and grouping columns and verify query plans with EXPLAIN to keep reports responsive.
