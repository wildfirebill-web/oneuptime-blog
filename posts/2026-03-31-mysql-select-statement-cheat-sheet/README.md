# MySQL SELECT Statement Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SELECT, Query, Cheat Sheet

Description: A practical cheat sheet covering MySQL SELECT syntax, filtering, sorting, limiting, aliasing, and common clauses for everyday query writing.

---

## Basic Syntax

```sql
SELECT column1, column2
FROM   table_name
WHERE  condition
ORDER BY column1 ASC
LIMIT  10 OFFSET 0;
```

## Select All Columns

```sql
SELECT * FROM employees;
```

## Column Aliases

```sql
SELECT first_name AS fname,
       last_name  AS lname,
       salary * 12 AS annual_salary
FROM employees;
```

## Distinct Values

```sql
SELECT DISTINCT department_id FROM employees;
```

## Filtering with WHERE

```sql
-- Comparison operators
SELECT * FROM orders WHERE total > 100;
SELECT * FROM orders WHERE status = 'PENDING';

-- Multiple conditions
SELECT * FROM orders WHERE status = 'SHIPPED' AND total > 500;
SELECT * FROM orders WHERE status = 'PENDING' OR status = 'PROCESSING';

-- Range
SELECT * FROM orders WHERE total BETWEEN 100 AND 500;

-- List membership
SELECT * FROM orders WHERE status IN ('PENDING', 'SHIPPED', 'DELIVERED');

-- Pattern matching
SELECT * FROM customers WHERE email LIKE '%@gmail.com';

-- NULL checks
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE manager_id IS NOT NULL;
```

## Sorting Results

```sql
SELECT name, salary FROM employees ORDER BY salary DESC;

-- Multiple sort keys
SELECT name, dept, salary
FROM employees
ORDER BY dept ASC, salary DESC;
```

## Limiting Results

```sql
-- First 10 rows
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- Rows 21-30 (pagination)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10 OFFSET 20;
```

## Grouping and Aggregates

```sql
SELECT department_id,
       COUNT(*)       AS headcount,
       AVG(salary)    AS avg_salary,
       MAX(salary)    AS max_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 50000;
```

## Subqueries in SELECT

```sql
-- Scalar subquery
SELECT name,
       salary,
       (SELECT AVG(salary) FROM employees) AS company_avg
FROM employees;

-- Subquery in WHERE
SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

## CASE Expressions

```sql
SELECT name,
       salary,
       CASE
         WHEN salary >= 100000 THEN 'High'
         WHEN salary >= 60000  THEN 'Mid'
         ELSE 'Low'
       END AS salary_band
FROM employees;
```

## Common Clauses Reference

```text
SELECT   - columns or expressions to return
FROM     - source table(s)
JOIN     - combine rows from other tables
WHERE    - filter rows before grouping
GROUP BY - aggregate rows by one or more columns
HAVING   - filter groups after aggregation
ORDER BY - sort result rows
LIMIT    - cap number of returned rows
OFFSET   - skip N rows before returning results
```

## Using SQL_CALC_FOUND_ROWS (Legacy Pattern)

```sql
SELECT SQL_CALC_FOUND_ROWS * FROM products LIMIT 10;
SELECT FOUND_ROWS();  -- total matching rows ignoring LIMIT
```

Note: prefer `COUNT(*)` subqueries in MySQL 8.x for better optimizer support.

## Summary

MySQL's SELECT statement is the cornerstone of data retrieval. Use WHERE for row filtering, GROUP BY with HAVING for aggregations, ORDER BY for sorting, and LIMIT/OFFSET for pagination. CASE expressions add conditional logic inline, while subqueries let you compose complex queries without application-side processing.
