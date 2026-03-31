# How to Generate Quarterly Reports in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Report, QUARTER, GROUP BY, Aggregation

Description: Learn how to generate quarterly reports in MySQL using the QUARTER() function, CASE-based grouping, and quarter-over-quarter LAG comparisons for business analytics.

---

## Grouping Data by Quarter

MySQL's `QUARTER()` function returns the quarter number (1-4) for a given date:

```sql
SELECT
  YEAR(order_date)    AS year,
  QUARTER(order_date) AS quarter,
  COUNT(*)            AS order_count,
  SUM(total_amount)   AS quarterly_revenue,
  AVG(total_amount)   AS avg_order_value
FROM orders
WHERE order_date >= CURDATE() - INTERVAL 2 YEAR
GROUP BY YEAR(order_date), QUARTER(order_date)
ORDER BY year, quarter;
```

## Labeling Quarters

Add a human-readable quarter label:

```sql
SELECT
  CONCAT(YEAR(order_date), '-Q', QUARTER(order_date)) AS quarter_label,
  COUNT(*)          AS orders,
  SUM(total_amount) AS revenue
FROM orders
GROUP BY quarter_label
ORDER BY quarter_label;
```

## Quarter Start and End Dates

Calculate the first and last day of each quarter for reporting:

```sql
SELECT
  YEAR(order_date)    AS yr,
  QUARTER(order_date) AS q,
  MAKEDATE(YEAR(order_date), 1)
    + INTERVAL (QUARTER(order_date) - 1) * 3 MONTH AS quarter_start,
  LAST_DAY(MAKEDATE(YEAR(order_date), 1)
    + INTERVAL QUARTER(order_date) * 3 - 1 MONTH)  AS quarter_end,
  SUM(total_amount) AS revenue
FROM orders
GROUP BY yr, q
ORDER BY yr, q;
```

## Quarter-over-Quarter Comparison

Use `LAG()` to compare each quarter against the previous:

```sql
WITH quarterly_revenue AS (
  SELECT
    YEAR(order_date)    AS yr,
    QUARTER(order_date) AS q,
    SUM(total_amount)   AS revenue
  FROM orders
  GROUP BY yr, q
)
SELECT
  yr,
  q,
  revenue,
  LAG(revenue) OVER (ORDER BY yr, q) AS prev_quarter_revenue,
  ROUND(
    100.0 * (revenue - LAG(revenue) OVER (ORDER BY yr, q))
    / NULLIF(LAG(revenue) OVER (ORDER BY yr, q), 0),
    2
  ) AS qoq_growth_pct
FROM quarterly_revenue
ORDER BY yr, q;
```

## Quarterly Report by Product Line

Break down quarterly revenue by product category:

```sql
SELECT
  YEAR(o.order_date)                              AS yr,
  QUARTER(o.order_date)                           AS q,
  p.category,
  SUM(oi.quantity * oi.unit_price)                AS revenue,
  COUNT(DISTINCT o.customer_id)                   AS customers
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
GROUP BY yr, q, p.category
ORDER BY yr, q, revenue DESC;
```

## Quarterly KPI Summary

Include multiple KPIs in a single quarterly summary:

```sql
SELECT
  CONCAT(YEAR(o.order_date), '-Q', QUARTER(o.order_date)) AS quarter,
  COUNT(DISTINCT o.id)                                     AS total_orders,
  COUNT(DISTINCT o.customer_id)                            AS active_customers,
  SUM(o.total_amount)                                      AS gross_revenue,
  SUM(CASE WHEN o.status = 'cancelled' THEN o.total_amount ELSE 0 END) AS cancelled_revenue,
  SUM(o.total_amount)
    - SUM(CASE WHEN o.status = 'cancelled' THEN o.total_amount ELSE 0 END) AS net_revenue
FROM orders o
GROUP BY quarter
ORDER BY quarter;
```

## Summary

Generate quarterly reports in MySQL using `YEAR()` and `QUARTER()` in a compound `GROUP BY`. Add human-readable labels with `CONCAT`. Use `LAG()` for quarter-over-quarter growth, and break down by product category or customer segment for detailed analysis. Pre-aggregate results into a quarterly report table for fast dashboard queries.
