# How to Use the AND Operator in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, AND Operator, WHERE Clause

Description: Learn how to use the MySQL AND operator to combine multiple conditions in WHERE clauses, including precedence rules and common filtering patterns.

---

The `AND` operator in MySQL combines two boolean conditions and returns TRUE only when both conditions are TRUE. It is used extensively in `WHERE`, `HAVING`, and `JOIN ... ON` clauses to apply multiple filters simultaneously.

## Basic AND Usage

```sql
-- Find active employees in the Engineering department
SELECT employee_id, name, department, status
FROM employees
WHERE department = 'Engineering'
  AND status = 'active';
```

Both conditions must be true for a row to be included in the results.

## Multiple AND Conditions

```sql
-- Find in-stock products in a specific price and category range
SELECT product_name, price, stock_qty, category
FROM products
WHERE category = 'Electronics'
  AND price >= 50.00
  AND price <= 500.00
  AND stock_qty > 0;
```

You can chain as many `AND` conditions as needed. All must evaluate to TRUE.

## AND in JOIN Conditions

```sql
-- Join with multiple conditions
SELECT o.order_id, o.total, c.name
FROM orders o
INNER JOIN customers c
  ON o.customer_id = c.customer_id
  AND c.status = 'active'
WHERE o.order_date >= '2025-01-01';
```

Filtering in the `ON` clause (versus `WHERE`) matters for outer joins - it controls whether rows are excluded or retained with NULLs.

## AND with Date Ranges

```sql
-- Find events in a date/time window
SELECT event_id, event_name, starts_at
FROM events
WHERE starts_at >= '2025-06-01 00:00:00'
  AND starts_at < '2025-07-01 00:00:00'
  AND is_cancelled = 0;
```

## Operator Precedence: AND vs OR

`AND` has higher precedence than `OR`. This means MySQL evaluates `AND` conditions before `OR` conditions:

```sql
-- This filters: (status = 'active' AND dept = 'Engineering') OR dept = 'Management'
SELECT * FROM employees
WHERE status = 'active'
  AND department = 'Engineering'
  OR department = 'Management';

-- Use parentheses to make intent clear
SELECT * FROM employees
WHERE status = 'active'
  AND (department = 'Engineering' OR department = 'Management');
```

Always use parentheses when mixing `AND` and `OR` to avoid ambiguity.

## AND with NULL

When any operand is NULL, the result of `AND` follows three-valued logic:

```sql
-- TRUE AND NULL = NULL (not included in results)
-- FALSE AND NULL = FALSE
-- NULL AND NULL = NULL
SELECT NULL AND TRUE;   -- Returns NULL
SELECT FALSE AND NULL;  -- Returns 0 (FALSE)
```

This means rows where a condition evaluates to NULL are excluded from results - the same as FALSE behavior.

## AND in the HAVING Clause

```sql
-- Find departments with more than 5 employees AND avg salary above 60000
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING headcount > 5
  AND avg_salary > 60000;
```

## Short-Circuit Evaluation

MySQL evaluates AND conditions left to right and stops as soon as it finds a FALSE condition. Place the most selective (and cheapest) condition first:

```sql
-- status check is fast and selective, put it first
SELECT * FROM orders
WHERE status = 'pending'
  AND YEAR(order_date) = 2025
  AND total > 1000;
```

## Summary

The `AND` operator requires all combined conditions to be TRUE for a row to pass the filter. It is used in `WHERE`, `HAVING`, and `JOIN ON` clauses. Remember that `AND` has higher precedence than `OR`, so use parentheses when mixing them. NULL propagation means conditions involving NULL values evaluate to NULL (not TRUE), effectively excluding those rows. Place cheap, selective conditions first for best performance.
