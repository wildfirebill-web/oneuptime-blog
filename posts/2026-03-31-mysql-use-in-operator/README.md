# How to Use the IN Operator in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, Operator, WHERE Clause, Filter

Description: Learn how to use the IN operator in MySQL to filter rows matching any value in a list, and understand its performance characteristics.

---

## What Is the IN Operator

The `IN` operator tests whether a column value matches any value in a provided list. It is a concise alternative to multiple `OR` conditions and works with literal values, subqueries, and expressions.

```sql
-- Without IN
SELECT * FROM products WHERE category = 'Books' OR category = 'Music' OR category = 'Movies';

-- With IN - cleaner and equivalent
SELECT * FROM products WHERE category IN ('Books', 'Music', 'Movies');
```

## Basic Syntax

```sql
SELECT column1, column2
FROM table_name
WHERE column_name IN (value1, value2, value3, ...);
```

## Filtering by a List of IDs

A common use case is retrieving specific rows by primary key:

```sql
SELECT id, username, email
FROM users
WHERE id IN (1, 5, 12, 47, 103);
```

## Using IN with Strings

```sql
SELECT order_id, status, amount
FROM orders
WHERE status IN ('pending', 'processing', 'shipped');
```

## Using NOT IN to Exclude Values

The `NOT IN` operator excludes rows matching any value in the list:

```sql
SELECT id, name
FROM products
WHERE category NOT IN ('Discontinued', 'Draft');
```

Be careful with `NOT IN` when the list may contain `NULL`. If any value in the list is `NULL`, the entire `NOT IN` expression returns `NULL` (not `TRUE`), and no rows are returned:

```sql
-- Dangerous: returns no rows if NULL is in the subquery result
SELECT * FROM orders WHERE user_id NOT IN (SELECT id FROM banned_users);

-- Safer alternative using NOT EXISTS
SELECT o.*
FROM orders o
WHERE NOT EXISTS (
  SELECT 1 FROM banned_users b WHERE b.id = o.user_id
);
```

## IN with a Subquery

The `IN` operator can compare against a subquery result set:

```sql
SELECT name, department
FROM employees
WHERE department_id IN (
  SELECT id FROM departments WHERE location = 'New York'
);
```

## Performance Considerations

For small lists, `IN` is efficient and readable. For large lists or subqueries, consider:

- Ensuring the column being tested is indexed
- Using `JOIN` instead of a subquery for better optimizer control

```sql
-- Often faster than IN with a subquery for large datasets
SELECT e.name, e.department
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE d.location = 'New York';
```

Check whether the index is used with `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM orders WHERE status IN ('pending', 'processing');
```

## Summary

The `IN` operator simplifies queries that filter against multiple values, replacing chains of `OR` conditions. It works with literal lists and subqueries alike. Avoid `NOT IN` when the list may contain `NULL` values; use `NOT EXISTS` as a safer alternative in those cases.
