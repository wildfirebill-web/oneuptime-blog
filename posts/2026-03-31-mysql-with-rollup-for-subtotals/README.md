# How to Use WITH ROLLUP for Subtotals in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ROLLUP, Subtotal, GROUP BY, Report

Description: Learn how to use WITH ROLLUP in MySQL to add automatic subtotals and grand totals to any GROUP BY query, with GROUPING() for labeling rollup rows.

---

## WITH ROLLUP Overview

`WITH ROLLUP` is a MySQL modifier for `GROUP BY` that automatically appends subtotal rows for each grouping level. It is the standard way to produce hierarchical totals without writing multiple aggregation queries and combining them with `UNION ALL`.

## Adding a Grand Total Row

The simplest use - add a single grand total row to a grouped query:

```sql
SELECT
  COALESCE(status, 'GRAND TOTAL') AS status,
  COUNT(*)                         AS order_count,
  SUM(total_amount)                AS revenue
FROM orders
GROUP BY status WITH ROLLUP;
```

The grand total row has `NULL` for the `status` column. `COALESCE` replaces it with a label.

## Two-Level Subtotals

For a region-and-status breakdown, ROLLUP adds subtotals per region plus a grand total:

```sql
SELECT
  COALESCE(region, 'All Regions')   AS region,
  COALESCE(status, 'Region Total')  AS status,
  COUNT(*)                          AS orders,
  SUM(total_amount)                 AS revenue
FROM orders
GROUP BY region, status WITH ROLLUP
ORDER BY region, status;
```

## Using GROUPING() to Identify Rollup Rows

`GROUPING(column)` returns 1 for a ROLLUP-generated row and 0 for a real data row:

```sql
SELECT
  CASE GROUPING(region) WHEN 1 THEN 'Grand Total' ELSE region END AS region,
  CASE GROUPING(status) WHEN 1 THEN 'Subtotal'    ELSE status END AS status,
  COUNT(*)              AS orders,
  SUM(total_amount)     AS revenue,
  GROUPING(region)      AS is_region_rollup,
  GROUPING(status)      AS is_status_rollup
FROM orders
GROUP BY region, status WITH ROLLUP;
```

## Styling Subtotal Rows

Mark subtotal rows for conditional formatting in reports:

```sql
SELECT
  IF(GROUPING(p.category), 'TOTAL', p.category)     AS category,
  IF(GROUPING(YEAR(o.order_date)), '', YEAR(o.order_date)) AS year,
  SUM(oi.quantity * oi.unit_price)                   AS revenue,
  IF(GROUPING(p.category) OR GROUPING(YEAR(o.order_date)), 1, 0) AS is_subtotal
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
GROUP BY p.category, YEAR(o.order_date) WITH ROLLUP;
```

## ROLLUP vs Manual UNION ALL

Without ROLLUP, producing the same result requires separate queries:

```sql
-- Without ROLLUP (verbose)
SELECT status, COUNT(*) AS cnt, SUM(total_amount) AS rev FROM orders GROUP BY status
UNION ALL
SELECT 'GRAND TOTAL', COUNT(*), SUM(total_amount) FROM orders;

-- With ROLLUP (concise)
SELECT COALESCE(status, 'GRAND TOTAL'), COUNT(*), SUM(total_amount)
FROM orders
GROUP BY status WITH ROLLUP;
```

## ROLLUP for Financial Summaries

Apply to a sales hierarchy:

```sql
SELECT
  IF(GROUPING(region),     'All Regions',     region)     AS region,
  IF(GROUPING(department), 'All Departments', department) AS dept,
  IF(GROUPING(salesperson),'Team Total',      salesperson) AS rep,
  SUM(deal_value) AS total_deals,
  COUNT(*)        AS deal_count
FROM sales
GROUP BY region, department, salesperson WITH ROLLUP
ORDER BY region, department, salesperson;
```

## Summary

`WITH ROLLUP` appends one subtotal row per `GROUP BY` level, plus a grand total row, all in a single efficient query. Use `COALESCE` or `GROUPING()` to identify and label rollup rows. For two-level hierarchies, ROLLUP adds a subtotal after each group and a grand total at the end - eliminating the need for `UNION ALL` patterns.
