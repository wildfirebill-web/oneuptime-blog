# How to Use ORDER BY with Multiple Columns in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ORDER BY, Sorting

Description: Learn how to sort MySQL query results by multiple columns using ORDER BY, controlling sort direction per column and understanding tie-breaking behavior.

---

Sorting by a single column is often insufficient when rows share the same value in that column. MySQL's `ORDER BY` clause accepts multiple columns separated by commas, applying each as a tie-breaker for the previous sort level.

## Basic Multi-Column Sorting

```sql
-- Sort by department first, then by salary descending within each department
SELECT employee_id, name, department, salary
FROM employees
ORDER BY department ASC, salary DESC;
```

MySQL first sorts all rows by `department` alphabetically. Within each department, rows with the same department are further sorted by `salary` in descending order.

## Independent Sort Directions Per Column

Each column in `ORDER BY` can have its own `ASC` or `DESC` direction:

```sql
-- Priority tasks first (low number = high priority), then by due date ascending
SELECT task_id, title, priority, due_date
FROM tasks
WHERE status = 'open'
ORDER BY priority ASC, due_date ASC;

-- Latest orders first, then alphabetically by customer name
SELECT order_id, customer_name, order_date
FROM orders
ORDER BY order_date DESC, customer_name ASC;
```

## Three-Level Sort

```sql
-- Sort by country, then state, then city
SELECT customer_id, name, city, state, country
FROM customers
ORDER BY country ASC, state ASC, city ASC;
```

MySQL processes each sort level left to right. `country` is the primary sort, `state` is secondary, and `city` is tertiary.

## Sorting by Expression

```sql
-- Sort by full name (computed) and then by signup date
SELECT user_id,
       CONCAT(last_name, ', ', first_name) AS full_name,
       signup_date
FROM users
ORDER BY full_name ASC, signup_date DESC;
```

## Using Column Position Numbers

You can reference columns by their position in the SELECT list:

```sql
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
ORDER BY 3 DESC, 1 ASC;
-- Same as: ORDER BY avg_salary DESC, department ASC
```

Using positional references works but makes queries harder to read. Column names or aliases are preferred.

## Performance with Indexes

A composite index can serve multi-column ORDER BY without a sort operation:

```sql
-- This index can serve ORDER BY (department ASC, salary DESC) in MySQL 8+
CREATE INDEX idx_dept_salary ON employees (department ASC, salary DESC);

-- Verify no filesort is used
EXPLAIN SELECT employee_id, name, department, salary
FROM employees
ORDER BY department ASC, salary DESC\G
```

In older MySQL versions, mixed ASC/DESC on a composite index required a filesort. MySQL 8.0 introduced descending index support, allowing the optimizer to use the index for mixed-direction multi-column sorts.

## NULL Ordering

In MySQL, NULL values sort before any non-NULL value in `ASC` order (and last in `DESC` order). To control NULL placement:

```sql
-- Put NULLs last in ascending order
SELECT task_id, title, due_date
FROM tasks
ORDER BY (due_date IS NULL) ASC, due_date ASC;
```

## Summary

Multi-column `ORDER BY` applies each column as a successive tie-breaker. Each column can have its own `ASC` or `DESC` direction independently. For best performance, create a composite index that matches the column order and direction of your `ORDER BY` clause - MySQL 8.0+ supports indexes with mixed sort directions. When rows contain NULLs, be explicit about where you want NULLs to appear, since MySQL defaults to placing them first in ascending sorts.
