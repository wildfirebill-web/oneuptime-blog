# How to Use UNION in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Union, Set Operations, Sql, Database

Description: Learn how to use UNION in MySQL to combine results from multiple SELECT statements into a single result set, automatically removing duplicate rows.

---

## Introduction

`UNION` in MySQL combines the result sets of two or more `SELECT` statements into a single result. By default, `UNION` removes duplicate rows from the combined output. This makes it useful for merging data from similar tables or queries while ensuring uniqueness.

## Basic UNION Syntax

```sql
SELECT column1, column2 FROM table1
UNION
SELECT column1, column2 FROM table2;
```

Rules:
- Each SELECT must have the same number of columns.
- Corresponding columns must have compatible data types.
- Column names in the result come from the first SELECT.

## Simple UNION Example

Combine customers from two regional tables:

```sql
SELECT id, name, email FROM customers_us
UNION
SELECT id, name, email FROM customers_eu;
```

Duplicate rows (same id, name, email) are automatically removed.

## UNION with Different Tables

Merge employees and contractors into one list:

```sql
SELECT name, 'Employee' AS type FROM employees
UNION
SELECT name, 'Contractor' AS type FROM contractors;
```

## UNION vs UNION ALL

- `UNION` removes duplicates (slower, requires deduplication).
- `UNION ALL` keeps all rows including duplicates (faster).

```sql
-- UNION removes duplicates
SELECT email FROM subscribers
UNION
SELECT email FROM newsletter_list;

-- UNION ALL keeps all rows
SELECT email FROM subscribers
UNION ALL
SELECT email FROM newsletter_list;
```

## UNION with Three or More Queries

```sql
SELECT name, 'Active' AS status FROM active_users
UNION
SELECT name, 'Inactive' AS status FROM inactive_users
UNION
SELECT name, 'Pending' AS status FROM pending_users;
```

## UNION with WHERE Filters

Each SELECT in a UNION can have its own WHERE clause.

```sql
SELECT product_id, name FROM products WHERE category = 'Electronics'
UNION
SELECT product_id, name FROM archived_products WHERE category = 'Electronics';
```

## UNION with ORDER BY and LIMIT

`ORDER BY` and `LIMIT` at the end apply to the entire union result.

```sql
SELECT name, salary FROM employees WHERE department = 'Engineering'
UNION
SELECT name, salary FROM contractors WHERE department = 'Engineering'
ORDER BY salary DESC
LIMIT 10;
```

Note: You cannot have `ORDER BY` in individual SELECT statements within a UNION (except when combined with LIMIT to influence subquery behavior).

## Using Parentheses for Clarity

Use parentheses to apply ORDER BY or LIMIT to individual parts:

```sql
(SELECT name, hire_date FROM employees ORDER BY hire_date DESC LIMIT 5)
UNION
(SELECT name, hire_date FROM contractors ORDER BY hire_date DESC LIMIT 5)
ORDER BY hire_date DESC;
```

## UNION with GROUP BY

Each SELECT can have its own GROUP BY:

```sql
SELECT department, SUM(salary) AS total FROM employees GROUP BY department
UNION
SELECT department, SUM(rate * hours) AS total FROM contractors GROUP BY department;
```

## Adding a Source Column

To know which table each row came from, add a literal column:

```sql
SELECT id, name, 'employees' AS source FROM employees
UNION
SELECT id, name, 'contractors' AS source FROM contractors;
```

## UNION to Simulate FULL OUTER JOIN

MySQL does not support FULL OUTER JOIN directly. Simulate it using UNION:

```sql
SELECT a.id, a.name, b.score
FROM table_a a LEFT JOIN table_b b ON a.id = b.id

UNION

SELECT a.id, a.name, b.score
FROM table_a a RIGHT JOIN table_b b ON a.id = b.id;
```

## Performance Considerations

`UNION` performs a deduplication step using a temporary table or sort, which adds overhead. If you do not need deduplication, use `UNION ALL` for better performance.

```sql
EXPLAIN
SELECT email FROM list1
UNION
SELECT email FROM list2;
```

Check for "Using temporary" in the Extra column.

## Summary

`UNION` combines multiple SELECT results into one result set and removes duplicate rows. It requires matching column counts and compatible types. Use `UNION ALL` when duplicates are acceptable or desired for better performance. Apply `ORDER BY` and `LIMIT` after the final SELECT to sort or limit the combined result.
