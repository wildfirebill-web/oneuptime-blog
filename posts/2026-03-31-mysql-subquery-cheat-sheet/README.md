# MySQL Subquery Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Subquery, Query, Cheat Sheet

Description: A practical cheat sheet for MySQL subqueries covering scalar, correlated, derived table, EXISTS, IN, and ANY/ALL patterns with ready-to-use examples.

---

## Types of Subqueries

```text
Scalar subquery    - returns a single value
Row subquery       - returns a single row
Table subquery     - returns multiple rows and columns (derived table)
Correlated         - references columns from the outer query
```

## Scalar Subquery in SELECT

```sql
SELECT name,
       salary,
       (SELECT AVG(salary) FROM employees) AS avg_sal
FROM employees;
```

## Subquery in WHERE with Comparison

```sql
-- Employees earning above average
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

## Subquery with IN

```sql
-- Orders placed by VIP customers
SELECT * FROM orders
WHERE customer_id IN (
  SELECT id FROM customers WHERE tier = 'VIP'
);
```

## Subquery with NOT IN (watch for NULLs)

```sql
-- Customers with no orders
SELECT name FROM customers
WHERE id NOT IN (
  SELECT DISTINCT customer_id FROM orders
  WHERE customer_id IS NOT NULL
);
```

## EXISTS (correlated, stops at first match)

```sql
-- Customers who have at least one order
SELECT c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

## NOT EXISTS

```sql
-- Products never ordered
SELECT p.name
FROM products p
WHERE NOT EXISTS (
  SELECT 1 FROM order_items oi WHERE oi.product_id = p.id
);
```

## Correlated Subquery

Executes once per outer row - can be slow on large tables; use with care.

```sql
-- Each employee's salary vs. their department average
SELECT name,
       salary,
       (SELECT AVG(salary)
        FROM employees e2
        WHERE e2.dept_id = e1.dept_id) AS dept_avg
FROM employees e1;
```

## Derived Table (inline view)

```sql
-- Average order value per customer
SELECT c.name, avg_orders.avg_val
FROM customers c
INNER JOIN (
  SELECT customer_id, AVG(total) AS avg_val
  FROM orders
  GROUP BY customer_id
) avg_orders ON avg_orders.customer_id = c.id;
```

## ANY / ALL

```sql
-- Salary higher than any employee in dept 5
SELECT name, salary
FROM employees
WHERE salary > ANY (
  SELECT salary FROM employees WHERE dept_id = 5
);

-- Salary higher than all employees in dept 5
SELECT name, salary
FROM employees
WHERE salary > ALL (
  SELECT salary FROM employees WHERE dept_id = 5
);
```

## Subquery in FROM with Aggregation

```sql
SELECT dept_id, MAX(total_salary) AS max_dept_salary
FROM (
  SELECT dept_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY dept_id
) dept_totals
GROUP BY dept_id;
```

## Subquery vs CTE (when to prefer each)

```text
Subquery  - simple one-off use, inline logic
CTE       - reused logic, recursive queries, readability
```

```sql
-- Same query as CTE
WITH dept_totals AS (
  SELECT dept_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY dept_id
)
SELECT dept_id, total_salary FROM dept_totals;
```

## Summary

MySQL subqueries appear in SELECT, FROM, and WHERE clauses. Scalar subqueries return one value; derived tables return a full result set usable as a virtual table. EXISTS is more efficient than IN for correlated lookups since it short-circuits at the first match. For complex or reused subquery logic, prefer CTEs for clarity.
