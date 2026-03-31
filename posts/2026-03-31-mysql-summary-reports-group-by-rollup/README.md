# How to Create Summary Reports with GROUP BY and ROLLUP in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ROLLUP, GROUP BY, Report, Aggregation

Description: Learn how to create hierarchical summary reports in MySQL using GROUP BY with ROLLUP to produce subtotals and grand totals in a single query without UNION ALL.

---

## What Is GROUP BY with ROLLUP?

`WITH ROLLUP` extends `GROUP BY` to produce subtotals at each grouping level and a grand total at the end. It eliminates the need for multiple `UNION ALL` queries to combine detail rows with summary rows.

## Basic ROLLUP Example

Generate revenue by year and quarter, with yearly subtotals and a grand total:

```sql
SELECT
  YEAR(order_date)    AS year,
  QUARTER(order_date) AS quarter,
  SUM(total_amount)   AS revenue
FROM orders
GROUP BY YEAR(order_date), QUARTER(order_date) WITH ROLLUP
ORDER BY year, quarter;
```

Result includes:
- Detail rows: each year/quarter combination
- Subtotals: each year with NULL quarter (yearly total)
- Grand total: NULL year, NULL quarter

## Labeling Subtotal Rows with GROUPING()

`GROUPING()` returns 1 when a column is aggregated by ROLLUP (i.e., represents a subtotal), letting you replace NULL with descriptive labels:

```sql
SELECT
  IF(GROUPING(YEAR(order_date)), 'All Years', YEAR(order_date))         AS year,
  IF(GROUPING(QUARTER(order_date)), 'All Quarters', QUARTER(order_date)) AS quarter,
  SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date), QUARTER(order_date) WITH ROLLUP;
```

## Three-Level Hierarchy

Category - Year - Quarter with subtotals at each level:

```sql
SELECT
  IF(GROUPING(p.category), 'ALL CATEGORIES', p.category)    AS category,
  IF(GROUPING(YEAR(o.order_date)), 'All Years', YEAR(o.order_date)) AS year,
  IF(GROUPING(QUARTER(o.order_date)), 'All Quarters', QUARTER(o.order_date)) AS quarter,
  SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
GROUP BY p.category, YEAR(o.order_date), QUARTER(o.order_date) WITH ROLLUP
ORDER BY p.category, year, quarter;
```

## Counting Distinct Values with ROLLUP

ROLLUP works with any aggregate function:

```sql
SELECT
  IF(GROUPING(region), 'All Regions', region)     AS region,
  IF(GROUPING(status), 'All Statuses', status)     AS status,
  COUNT(*)                                         AS order_count,
  SUM(total_amount)                                AS revenue,
  COUNT(DISTINCT customer_id)                      AS customers
FROM orders
GROUP BY region, status WITH ROLLUP
ORDER BY region, status;
```

## Filtering Out Grand Total

If you only want the subtotals but not the grand total:

```sql
SELECT
  IF(GROUPING(YEAR(order_date)), 'Total', CAST(YEAR(order_date) AS CHAR)) AS year,
  IF(GROUPING(QUARTER(order_date)), 'Year Total', CAST(QUARTER(order_date) AS CHAR)) AS quarter,
  SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(order_date), QUARTER(order_date) WITH ROLLUP
HAVING GROUPING(YEAR(order_date)) = 0  -- exclude grand total row
ORDER BY year, quarter;
```

## Summary

`GROUP BY ... WITH ROLLUP` generates hierarchical subtotals in a single MySQL query. Use `GROUPING()` to detect which rows are subtotals and replace NULL values with descriptive labels like "All Years" or "Total". This approach is more efficient than multiple `UNION ALL` queries and works with all standard aggregate functions.
