# How to Use the HAVING Clause in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Having, Group By, Aggregation, Sql

Description: Learn how to use the HAVING clause in MySQL to filter grouped results after GROUP BY, with examples comparing HAVING vs WHERE.

---

## Introduction

The `HAVING` clause in MySQL filters the results of a `GROUP BY` query. While `WHERE` filters individual rows before grouping, `HAVING` filters aggregated groups after grouping. This makes `HAVING` essential when you want to apply conditions on aggregate values like `COUNT()`, `SUM()`, or `AVG()`.

## Basic HAVING Syntax

```sql
SELECT column1, aggregate_function(column2)
FROM table_name
GROUP BY column1
HAVING condition;
```

## Simple HAVING Example

Find departments with more than 10 employees:

```sql
SELECT department, COUNT(*) AS employee_count
FROM employees
GROUP BY department
HAVING COUNT(*) > 10;
```

## HAVING vs WHERE

- `WHERE` filters rows before grouping (cannot use aggregate functions).
- `HAVING` filters groups after aggregation (can use aggregate functions).

```sql
-- WHERE filters rows before grouping
SELECT department, COUNT(*) AS count
FROM employees
WHERE status = 'active'
GROUP BY department;

-- HAVING filters groups after grouping
SELECT department, COUNT(*) AS count
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;

-- Both together
SELECT department, COUNT(*) AS count
FROM employees
WHERE status = 'active'
GROUP BY department
HAVING COUNT(*) > 5;
```

## HAVING with SUM

Find products categories where total revenue exceeds 50,000:

```sql
SELECT category, SUM(price * quantity) AS total_revenue
FROM order_items
GROUP BY category
HAVING SUM(price * quantity) > 50000
ORDER BY total_revenue DESC;
```

## HAVING with AVG

Find departments where the average salary is above 60,000:

```sql
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 60000;
```

## HAVING with MIN and MAX

Find customers who have placed orders where the maximum order value exceeds 1,000:

```sql
SELECT customer_id, MAX(total_amount) AS max_order
FROM orders
GROUP BY customer_id
HAVING MAX(total_amount) > 1000;
```

## HAVING with Column Alias

MySQL allows using column aliases from the SELECT list in HAVING (unlike standard SQL).

```sql
SELECT department, COUNT(*) AS emp_count
FROM employees
GROUP BY department
HAVING emp_count > 10;
```

Note: This is a MySQL extension. Other databases may not support this.

## HAVING with Multiple Conditions

Combine conditions using AND and OR.

```sql
SELECT department, COUNT(*) AS emp_count, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING emp_count > 5 AND avg_salary > 55000;
```

## HAVING with ORDER BY and LIMIT

```sql
SELECT department, SUM(salary) AS total_salary
FROM employees
GROUP BY department
HAVING total_salary > 100000
ORDER BY total_salary DESC
LIMIT 5;
```

## HAVING without GROUP BY

`HAVING` can be used without `GROUP BY`, treating the entire result set as a single group.

```sql
SELECT COUNT(*) AS total_employees
FROM employees
HAVING COUNT(*) > 100;
```

This returns the count only if there are more than 100 employees, otherwise returns no rows.

## Performance Considerations

- Use `WHERE` to filter rows early and reduce the data processed by `GROUP BY`.
- Index columns used in `GROUP BY` and `WHERE` for better performance.
- Avoid applying heavy calculations in `HAVING` if a `WHERE` filter can achieve the same result.

```sql
-- Less efficient: processes all rows then filters groups
SELECT status, COUNT(*) FROM orders GROUP BY status HAVING status = 'completed';

-- More efficient: filters rows before grouping
SELECT status, COUNT(*) FROM orders WHERE status = 'completed' GROUP BY status;
```

## Summary

`HAVING` is the correct way to filter aggregated groups in MySQL. Use `WHERE` for row-level filtering before grouping and `HAVING` for aggregate-level filtering after grouping. Combining both in the same query is common and efficient when done with proper indexing.
