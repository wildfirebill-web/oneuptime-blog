# How to Use the GROUP BY Clause in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Group By, Aggregation, Sql, Database

Description: Learn how to use the GROUP BY clause in MySQL to group rows by one or more columns and apply aggregate functions like COUNT, SUM, and AVG.

---

## Introduction

The `GROUP BY` clause in MySQL groups rows that share the same values in one or more columns into summary rows. It is almost always used with aggregate functions like `COUNT()`, `SUM()`, `AVG()`, `MIN()`, and `MAX()` to compute statistics per group.

## Basic GROUP BY Syntax

```sql
SELECT column1, aggregate_function(column2)
FROM table_name
GROUP BY column1;
```

Each unique value of `column1` becomes one row in the output.

## Counting Rows per Group

```sql
SELECT department, COUNT(*) AS employee_count
FROM employees
GROUP BY department;
```

Result:

```text
+------------+----------------+
| department | employee_count |
+------------+----------------+
| Engineering| 25             |
| Marketing  | 12             |
| HR         | 8              |
+------------+----------------+
```

## Summing Values per Group

```sql
SELECT department, SUM(salary) AS total_salary
FROM employees
GROUP BY department;
```

## Averaging Values per Group

```sql
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
```

## MIN and MAX per Group

```sql
SELECT department, MIN(salary) AS min_salary, MAX(salary) AS max_salary
FROM employees
GROUP BY department;
```

## GROUP BY with Multiple Columns

You can group by more than one column to create finer-grained groups.

```sql
SELECT department, job_title, COUNT(*) AS count
FROM employees
GROUP BY department, job_title
ORDER BY department, job_title;
```

## GROUP BY with WHERE

Use `WHERE` to filter rows before grouping. Note: `WHERE` runs before `GROUP BY`.

```sql
SELECT department, COUNT(*) AS employee_count
FROM employees
WHERE status = 'active'
GROUP BY department;
```

## GROUP BY with HAVING

Use `HAVING` to filter the grouped results. Unlike `WHERE`, `HAVING` runs after grouping.

```sql
SELECT department, COUNT(*) AS employee_count
FROM employees
GROUP BY department
HAVING employee_count > 10;
```

## GROUP BY with ORDER BY

Combine with `ORDER BY` to sort the grouped results.

```sql
SELECT department, SUM(salary) AS total_salary
FROM employees
GROUP BY department
ORDER BY total_salary DESC;
```

## GROUP BY with JOIN

You can group results from joined tables.

```sql
SELECT d.name AS department, COUNT(e.id) AS headcount
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name
ORDER BY headcount DESC;
```

## Using GROUP BY with DISTINCT

`GROUP BY` implicitly returns distinct values of the grouped columns, similar to `DISTINCT`. However, they serve different purposes - use `GROUP BY` when you also need aggregate functions.

```sql
-- These are equivalent for counting purposes
SELECT DISTINCT department FROM employees;
SELECT department FROM employees GROUP BY department;
```

## GROUP BY with Expressions

You can group by expressions, not just column names.

```sql
SELECT YEAR(order_date) AS year, COUNT(*) AS order_count
FROM orders
GROUP BY YEAR(order_date)
ORDER BY year;
```

## NULL Handling in GROUP BY

MySQL groups all NULL values together into a single group.

```sql
SELECT region, COUNT(*) AS count
FROM customers
GROUP BY region;
-- All rows with region = NULL are grouped together
```

## ONLY_FULL_GROUP_BY Mode

By default, MySQL 8 enforces `ONLY_FULL_GROUP_BY` mode, which means every non-aggregated column in `SELECT` must appear in `GROUP BY`.

```sql
-- This fails in ONLY_FULL_GROUP_BY mode
SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;

-- Fix: include name in GROUP BY or aggregate it
SELECT department, COUNT(*), MAX(name) AS sample_name
FROM employees
GROUP BY department;
```

## Performance Tips

- Index the columns you group by to speed up grouping operations.
- Filter with `WHERE` before `GROUP BY` to reduce the number of rows being grouped.
- Use `EXPLAIN` to check if an index is used:

```sql
EXPLAIN SELECT department, COUNT(*) FROM employees GROUP BY department;
```

## Summary

`GROUP BY` is essential for aggregating data in MySQL. It collapses multiple rows into summary rows based on shared column values and works with aggregate functions like COUNT, SUM, AVG, MIN, and MAX. Always remember that `WHERE` filters before grouping while `HAVING` filters after grouping.
