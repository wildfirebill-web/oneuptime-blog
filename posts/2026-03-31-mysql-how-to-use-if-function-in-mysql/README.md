# How to Use IF() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, IF Function, Conditional Logic, SQL Functions

Description: Learn how to use the IF() function in MySQL to implement conditional logic inline in SELECT, UPDATE, and other SQL statements.

---

## What is the IF() Function

MySQL's `IF()` function is an inline conditional expression that evaluates a condition and returns one of two values depending on whether the condition is true or false.

Syntax:

```sql
IF(condition, value_if_true, value_if_false)
```

It works like a ternary operator in programming languages and is useful for transforming data inline without needing CASE expressions.

## Basic Usage

```sql
-- Simple boolean condition
SELECT IF(1 > 0, 'Yes', 'No') AS result;
-- Output: Yes

SELECT IF(NULL = NULL, 'Equal', 'Not Equal') AS result;
-- Output: Not Equal (NULL comparisons return NULL, not TRUE)
```

## Using IF() in a SELECT

```sql
SELECT
  name,
  salary,
  IF(salary >= 90000, 'Senior', 'Junior') AS level
FROM employees;
```

Output:

```text
+-------------+----------+--------+
| name        | salary   | level  |
+-------------+----------+--------+
| Alice Smith | 95000.00 | Senior |
| Bob Jones   | 72000.00 | Junior |
+-------------+----------+--------+
```

## Nested IF() Expressions

You can nest `IF()` for multiple conditions:

```sql
SELECT
  name,
  salary,
  IF(salary >= 100000, 'L5',
    IF(salary >= 80000, 'L4',
      IF(salary >= 60000, 'L3', 'L2')
    )
  ) AS pay_band
FROM employees;
```

For readability with many conditions, prefer `CASE WHEN` over deeply nested `IF()`.

## IF() in UPDATE Statements

```sql
UPDATE employees
SET salary = IF(department = 'Engineering', salary * 1.10, salary * 1.05);
```

This gives Engineering a 10% raise and everyone else 5%.

## IF() with Aggregate Functions

A powerful use case is conditional aggregation:

```sql
SELECT
  department,
  COUNT(*) AS total,
  SUM(IF(salary >= 90000, 1, 0)) AS senior_count,
  SUM(IF(salary < 90000, 1, 0)) AS junior_count
FROM employees
GROUP BY department;
```

## IF() vs CASE WHEN

`IF()` is concise but limited to two outcomes. Use `CASE WHEN` for multiple conditions:

```sql
-- IF() - two outcomes
SELECT IF(status = 'active', 'Active', 'Inactive') FROM users;

-- CASE WHEN - multiple outcomes
SELECT
  CASE status
    WHEN 'active' THEN 'Active'
    WHEN 'suspended' THEN 'Suspended'
    WHEN 'deleted' THEN 'Deleted'
    ELSE 'Unknown'
  END AS status_label
FROM users;
```

## IF() Handling NULL Values

`IF()` does not special-case NULL. Use `IFNULL()` or `COALESCE()` for NULL handling:

```sql
-- IF() does NOT catch NULL in the condition
SELECT IF(NULL, 'True', 'False');
-- Output: False (NULL is treated as false)

-- To handle NULL in value
SELECT IF(name IS NULL, 'Unknown', name) AS display_name FROM employees;

-- Better: use IFNULL
SELECT IFNULL(name, 'Unknown') AS display_name FROM employees;
```

## Using IF() in ORDER BY

```sql
-- Sort active users first, then inactive
SELECT name, status
FROM users
ORDER BY IF(status = 'active', 0, 1), name;
```

## Summary

MySQL's `IF(condition, true_val, false_val)` provides a concise inline conditional expression for two-outcome logic in SELECT, UPDATE, and aggregate queries. Use it for simple branching and conditional aggregation. For three or more outcomes, prefer `CASE WHEN` for clarity. Remember that NULL conditions in `IF()` evaluate to false, so use `IS NULL` explicitly for null checks.
