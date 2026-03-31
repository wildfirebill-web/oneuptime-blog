# How to Use Inline Views in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, View

Description: Learn how to use inline views (subqueries in the FROM clause) in MySQL to simplify complex queries, pre-aggregate data, and build layered query logic.

---

An inline view is a subquery placed in the `FROM` clause of a query. MySQL treats it as a temporary, unnamed result set that the outer query can filter, join, and aggregate. Unlike stored views, inline views exist only for the duration of the query.

## Basic Inline View

The simplest use is pre-aggregating before joining:

```sql
SELECT c.name, orders.order_count
FROM customers c
JOIN (
  SELECT customer_id, COUNT(*) AS order_count
  FROM orders
  GROUP BY customer_id
) AS orders ON c.customer_id = orders.customer_id;
```

The subquery in the `FROM` clause computes per-customer order counts first, and the outer query joins the result to the customers table.

## Filtering on Aggregated Results

Inline views let you filter on aggregate values without `HAVING`:

```sql
SELECT department, avg_salary
FROM (
  SELECT department, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department
) AS dept_stats
WHERE avg_salary > 70000;
```

While this could also be written with `HAVING`, the inline view approach is useful when you need to reference the computed column name in downstream JOINs or further aggregations.

## Multi-Level Aggregation

Inline views allow you to aggregate already-aggregated data:

```sql
SELECT AVG(dept_avg) AS company_avg_salary
FROM (
  SELECT department, AVG(salary) AS dept_avg
  FROM employees
  GROUP BY department
) AS dept_averages;
```

This computes the average of department averages, which is different from the overall company average and requires the two-level query.

## Joining Multiple Inline Views

You can join several inline views together for complex reporting:

```sql
SELECT
  s.region,
  s.total_sales,
  r.total_returns,
  s.total_sales - r.total_returns AS net_sales
FROM
  (SELECT region, SUM(amount) AS total_sales   FROM sales   GROUP BY region) s
JOIN
  (SELECT region, SUM(amount) AS total_returns FROM returns GROUP BY region) r
  ON s.region = r.region;
```

## Ranking with Inline Views

Before MySQL 8.0 window functions were available, inline views with user variables were the way to add row numbers:

```sql
SELECT id, name, sales, rank_num
FROM (
  SELECT id, name, sales,
    @rownum := @rownum + 1 AS rank_num
  FROM employees, (SELECT @rownum := 0) AS init
  ORDER BY sales DESC
) AS ranked
WHERE rank_num <= 5;
```

In MySQL 8.0+, prefer the cleaner window function syntax instead.

## Inline Views vs CTEs

Both inline views and CTEs (Common Table Expressions) allow you to name intermediate results. CTEs use the `WITH` keyword and appear before the main query:

```sql
-- CTE approach
WITH dept_stats AS (
  SELECT department, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department
)
SELECT department, avg_salary
FROM dept_stats
WHERE avg_salary > 70000;
```

CTEs are more readable for complex logic because they separate the intermediate results from the main query. However, inline views are valid in all MySQL 5.x+ versions, while CTEs require MySQL 8.0+.

## Optimizer Behavior

MySQL may or may not materialize inline views depending on the optimizer. Use `EXPLAIN` to inspect the query plan:

```sql
EXPLAIN SELECT c.name, orders.order_count
FROM customers c
JOIN (
  SELECT customer_id, COUNT(*) AS order_count
  FROM orders
  GROUP BY customer_id
) AS orders ON c.customer_id = orders.customer_id;
```

Look for `DERIVED` in the `select_type` column, which indicates a materialized inline view.

## Summary

Inline views in MySQL are subqueries in the `FROM` clause that act as temporary result sets. They enable pre-aggregation, multi-level aggregation, and complex join scenarios. For MySQL 8.0+, CTEs are often a more readable alternative, but inline views remain essential for compatibility with older MySQL versions.
