# How to Sort Results with ORDER BY in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ORDER BY, Sorting, SQL, Database

Description: Learn how to sort query results using ORDER BY in MySQL, including ascending, descending, multi-column, and expression-based sorting.

---

## Introduction

The `ORDER BY` clause in MySQL sorts the rows returned by a `SELECT` statement. Without it, MySQL returns rows in an unpredictable order. You can sort by one or multiple columns, in ascending or descending order, and even by expressions or functions.

## Basic ORDER BY Syntax

```sql
SELECT column1, column2
FROM table_name
ORDER BY column1 [ASC | DESC];
```

By default, `ORDER BY` sorts in ascending order (`ASC`). Use `DESC` for descending order.

## Sorting in Ascending Order

```sql
SELECT id, name, salary
FROM employees
ORDER BY salary ASC;
```

This returns all employees sorted from lowest to highest salary.

## Sorting in Descending Order

```sql
SELECT id, name, salary
FROM employees
ORDER BY salary DESC;
```

This returns employees from highest to lowest salary.

## Sorting by Multiple Columns

You can sort by more than one column. MySQL applies the sort conditions left to right.

```sql
SELECT department, name, salary
FROM employees
ORDER BY department ASC, salary DESC;
```

This sorts first by department alphabetically, then within each department by salary from highest to lowest.

## Sorting by Column Position

You can reference columns by their position in the SELECT list instead of by name.

```sql
SELECT name, salary, department
FROM employees
ORDER BY 2 DESC;
```

Here, `2` refers to the `salary` column. This approach is convenient but less readable.

## Sorting with NULL Values

By default, MySQL treats NULL as less than any non-NULL value. NULLs appear first in ascending order and last in descending order.

```sql
SELECT name, bonus
FROM employees
ORDER BY bonus ASC;
-- NULLs appear first
```

To push NULLs to the end in ascending order, use a workaround:

```sql
SELECT name, bonus
FROM employees
ORDER BY bonus IS NULL ASC, bonus ASC;
```

## Sorting by Expression

You can sort by any valid expression or function result.

```sql
SELECT name, first_name, last_name
FROM employees
ORDER BY CONCAT(last_name, ' ', first_name) ASC;
```

Or sort by a computed value:

```sql
SELECT product_name, price, discount
FROM products
ORDER BY (price - discount) ASC;
```

## Sorting by String Functions

```sql
SELECT name
FROM customers
ORDER BY LENGTH(name) ASC;
```

This sorts names from shortest to longest.

## Sorting with CASE Expression

```sql
SELECT name, status
FROM orders
ORDER BY
  CASE status
    WHEN 'urgent'  THEN 1
    WHEN 'normal'  THEN 2
    WHEN 'low'     THEN 3
    ELSE 4
  END ASC;
```

This applies a custom sort order based on status priority.

## ORDER BY with LIMIT

Combining `ORDER BY` with `LIMIT` retrieves the top N rows.

```sql
SELECT name, salary
FROM employees
ORDER BY salary DESC
LIMIT 5;
```

This returns the 5 highest-paid employees.

## ORDER BY in Subqueries

Be careful with ORDER BY inside subqueries. MySQL may ignore it unless LIMIT is also specified.

```sql
SELECT *
FROM (
  SELECT id, name, salary
  FROM employees
  ORDER BY salary DESC
  LIMIT 10
) AS top_earners
ORDER BY name ASC;
```

## Performance Considerations

- If you frequently sort a column, adding an index on that column can speed up queries significantly.
- Sorting large result sets without an index triggers a filesort, which is slower.
- Use `EXPLAIN` to check if MySQL uses an index for sorting:

```sql
EXPLAIN SELECT name, salary FROM employees ORDER BY salary DESC;
```

Look for `Using filesort` in the Extra column - this means no index is being used for the sort.

## Summary

`ORDER BY` is the standard way to sort MySQL query results. You can sort by single or multiple columns, use ASC or DESC direction, and apply expressions or functions. For best performance on large tables, index the columns you sort by most frequently.
