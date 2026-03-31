# How to Use ASC and DESC in MySQL ORDER BY

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ORDER BY, ASC DESC

Description: Learn how to control sort direction in MySQL ORDER BY using ASC and DESC, including default behavior, mixed directions, and index optimization tips.

---

The `ORDER BY` clause in MySQL sorts query results. The `ASC` keyword sorts in ascending order (lowest to highest) and `DESC` sorts in descending order (highest to lowest). Understanding when and how to use each improves both query clarity and performance.

## Default Sort Direction

Without specifying a direction, MySQL defaults to `ASC`:

```sql
-- These two queries are equivalent
SELECT name, salary FROM employees ORDER BY salary;
SELECT name, salary FROM employees ORDER BY salary ASC;
```

Always being explicit with `ASC` or `DESC` makes queries easier to read and reason about.

## Ascending Order

```sql
-- Oldest orders first
SELECT order_id, order_date, total
FROM orders
ORDER BY order_date ASC;

-- Alphabetical by last name
SELECT customer_id, last_name, first_name
FROM customers
ORDER BY last_name ASC, first_name ASC;

-- Smallest to largest price
SELECT product_name, price
FROM products
ORDER BY price ASC;
```

## Descending Order

```sql
-- Most recent orders first
SELECT order_id, order_date, total
FROM orders
ORDER BY order_date DESC;

-- Highest earners first
SELECT name, department, salary
FROM employees
ORDER BY salary DESC;

-- Most viewed content first
SELECT title, view_count
FROM articles
ORDER BY view_count DESC;
```

## Mixed Directions on Multiple Columns

Each column in `ORDER BY` can have its own direction:

```sql
-- Highest priority first (1 = highest), then earliest due date
SELECT task_id, title, priority, due_date
FROM tasks
ORDER BY priority ASC, due_date ASC;

-- Department alphabetically, highest paid within each department
SELECT name, department, salary
FROM employees
ORDER BY department ASC, salary DESC;
```

## ASC and DESC with Expressions

```sql
-- Sort by calculated age, oldest first
SELECT name, birth_date,
       TIMESTAMPDIFF(YEAR, birth_date, CURDATE()) AS age
FROM users
ORDER BY age DESC;

-- Sort by month extracted from date, for seasonal analysis
SELECT order_id, order_date
FROM orders
ORDER BY MONTH(order_date) ASC;
```

## Indexes and Sort Direction

Standard B-tree indexes in MySQL store values in ascending order by default. MySQL 8.0 introduced true descending indexes:

```sql
-- MySQL 8.0+: create a descending index for frequently DESC-sorted columns
CREATE INDEX idx_created_desc ON posts (created_at DESC);

-- A composite index with mixed directions (MySQL 8.0+)
CREATE INDEX idx_dept_salary ON employees (department ASC, salary DESC);

-- Check if sort uses a filesort (bad) or index (good)
EXPLAIN SELECT name, department, salary
FROM employees
ORDER BY department ASC, salary DESC\G
```

If `EXPLAIN` shows `Using filesort` in the Extra column, MySQL is doing an in-memory or on-disk sort. Adding an index matching the ORDER BY columns and directions can eliminate the filesort.

## NULL Values in Sorted Results

In MySQL, NULL values sort first in `ASC` order and last in `DESC` order:

```sql
-- NULLs appear at the top with ASC
SELECT task_id, due_date FROM tasks ORDER BY due_date ASC;

-- NULLs appear at the bottom with DESC
SELECT task_id, due_date FROM tasks ORDER BY due_date DESC;

-- Force NULLs to appear last in ASC sort
SELECT task_id, due_date FROM tasks
ORDER BY (due_date IS NULL) ASC, due_date ASC;
```

## Summary

`ASC` sorts values from lowest to highest (alphabetically, numerically, or chronologically) and is the MySQL default. `DESC` sorts from highest to lowest. Each column in an `ORDER BY` clause can independently use `ASC` or `DESC`. For performance, create indexes that match your ORDER BY column order and direction - MySQL 8.0+ supports true descending indexes, enabling index-based sorting without filesort overhead.
