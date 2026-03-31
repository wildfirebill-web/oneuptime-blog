# How to Use GROUP BY with Multiple Columns in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, GROUP BY, Aggregate Function

Description: Learn how to use GROUP BY with multiple columns in MySQL to group data by compound keys, produce multi-dimensional summaries, and use ROLLUP for subtotals.

---

`GROUP BY` with multiple columns creates groups based on the unique combination of all specified columns. This is essential for building multi-dimensional reports, such as sales by region and year, or orders by status and customer tier.

## Basic Multi-Column GROUP BY

```sql
-- Count orders by status and year
SELECT
  YEAR(order_date) AS order_year,
  status,
  COUNT(*) AS order_count
FROM orders
GROUP BY order_year, status
ORDER BY order_year, status;
```

Each unique combination of `order_year` and `status` forms its own group. A row with year 2025 and status "completed" is a different group than 2025 and "pending".

## Revenue by Category and Month

```sql
SELECT
  YEAR(o.order_date) AS order_year,
  MONTH(o.order_date) AS order_month,
  p.category,
  SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
INNER JOIN orders o ON oi.order_id = o.order_id
INNER JOIN products p ON oi.product_id = p.product_id
GROUP BY order_year, order_month, p.category
ORDER BY order_year, order_month, p.category;
```

## GROUP BY with COUNT and SUM Together

```sql
-- Employees by department and job level: count and total salary
SELECT
  department,
  job_level,
  COUNT(*) AS headcount,
  SUM(salary) AS total_payroll,
  AVG(salary) AS avg_salary
FROM employees
GROUP BY department, job_level
ORDER BY department, job_level;
```

## HAVING with Multi-Column GROUP BY

```sql
-- Find department + job_level combinations with more than 3 employees
SELECT
  department,
  job_level,
  COUNT(*) AS headcount
FROM employees
GROUP BY department, job_level
HAVING headcount > 3
ORDER BY department, headcount DESC;
```

## ROLLUP for Subtotals

`WITH ROLLUP` adds subtotal and grand total rows automatically:

```sql
-- Revenue by year and category, with subtotals per year
SELECT
  COALESCE(CAST(YEAR(order_date) AS CHAR), 'All Years') AS order_year,
  COALESCE(category, 'All Categories') AS category,
  SUM(total) AS revenue
FROM orders o
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id
GROUP BY YEAR(order_date), category WITH ROLLUP
ORDER BY order_year, category;
```

`WITH ROLLUP` adds rows where individual columns are NULL representing subtotals. `COALESCE` converts those NULLs to readable labels.

## Grouping by Expressions

```sql
-- Group by quarter and region
SELECT
  QUARTER(order_date) AS quarter,
  region,
  COUNT(*) AS order_count,
  SUM(total) AS revenue
FROM orders
WHERE YEAR(order_date) = 2025
GROUP BY QUARTER(order_date), region
ORDER BY quarter, region;
```

## Using GROUPING() Function

In MySQL 8.0+ with `WITH ROLLUP`, use `GROUPING()` to distinguish NULL from rollup NULLs:

```sql
SELECT
  IF(GROUPING(department), 'All Departments', department) AS department,
  IF(GROUPING(job_level), 'All Levels', job_level) AS job_level,
  COUNT(*) AS headcount
FROM employees
GROUP BY department, job_level WITH ROLLUP;
```

## Performance Tips

```sql
-- Index the GROUP BY columns
CREATE INDEX idx_dept_level ON employees (department, job_level);

-- Check execution plan for temporary tables or filesort
EXPLAIN SELECT department, job_level, COUNT(*)
FROM employees
GROUP BY department, job_level\G
```

If `EXPLAIN` shows `Using temporary; Using filesort`, adding an index on the GROUP BY columns can eliminate both.

## Summary

Multi-column `GROUP BY` creates groups based on unique combinations of all specified columns. Aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) apply within each group. Use `HAVING` to filter groups after aggregation. `WITH ROLLUP` automatically adds subtotals and grand totals. Index the columns in your `GROUP BY` clause to avoid temporary tables and filesort operations on large tables.
