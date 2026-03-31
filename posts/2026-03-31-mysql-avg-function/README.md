# How to Use the AVG() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, AVG Function, Aggregate Function

Description: Learn how to use the MySQL AVG() function to compute average values, handle NULLs, compute averages per group, and use moving averages with window functions.

---

`AVG()` is a MySQL aggregate function that returns the arithmetic mean of non-NULL numeric values. It is commonly used to compute average order values, performance metrics, and statistical baselines.

## Basic AVG

```sql
-- Average order total across all orders
SELECT AVG(total) AS avg_order_value
FROM orders;

-- Average salary across all employees
SELECT AVG(salary) AS avg_salary FROM employees;
```

NULL values are excluded from the calculation. The average is computed as `SUM(non-null values) / COUNT(non-null values)`.

## AVG with WHERE

```sql
-- Average total for completed orders in 2025
SELECT AVG(total) AS avg_order_value
FROM orders
WHERE status = 'completed'
  AND YEAR(order_date) = 2025;
```

## AVG with GROUP BY

```sql
-- Average salary per department
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
ORDER BY avg_salary DESC;

-- Average order value per customer
SELECT customer_id, AVG(total) AS avg_order_value
FROM orders
GROUP BY customer_id
ORDER BY avg_order_value DESC;
```

## Rounding AVG Results

`AVG()` returns a decimal result. Use `ROUND()` to limit decimal places:

```sql
SELECT
  department,
  ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department;
```

## AVG with HAVING

```sql
-- Departments with average salary above $65,000
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING avg_salary > 65000
ORDER BY avg_salary DESC;
```

## Comparing Rows to the Group Average

```sql
-- Find employees earning above their department's average
SELECT
  e.name,
  e.department,
  e.salary,
  dept_avg.avg_salary
FROM employees e
INNER JOIN (
  SELECT department, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department
) AS dept_avg ON e.department = dept_avg.department
WHERE e.salary > dept_avg.avg_salary;
```

## Moving Average with AVG() Window Function

MySQL 8.0+ supports `AVG()` as a window function:

```sql
-- 7-day moving average of daily sales
SELECT
  sale_date,
  daily_total,
  ROUND(AVG(daily_total) OVER (
    ORDER BY sale_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ), 2) AS moving_avg_7d
FROM daily_sales
ORDER BY sale_date;
```

## AVG with NULL Handling

```sql
-- If some scores are NULL, AVG excludes them
SELECT AVG(test_score) AS avg_score FROM students;

-- To include NULLs as 0 in the average
SELECT AVG(IFNULL(test_score, 0)) AS avg_score_with_zeros FROM students;
```

The choice between these depends on your business logic - does a NULL score mean "not taken" (exclude) or "zero" (include as 0)?

## AVG of Expressions

```sql
-- Average line item value (quantity * unit_price)
SELECT AVG(quantity * unit_price) AS avg_line_value
FROM order_items;
```

## Summary

`AVG()` computes the mean of non-NULL values in a column, automatically ignoring NULLs. Use `ROUND()` to format decimal results. With `GROUP BY`, it computes per-group averages. Use `HAVING` to filter groups by their average. To treat NULLs as zero (changing the denominator), wrap the column in `IFNULL(col, 0)`. In MySQL 8.0+, `AVG() OVER (...)` enables moving averages and per-partition means without aggregating rows away.
