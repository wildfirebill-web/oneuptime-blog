# How to Use IFNULL() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Ifnull, Null Handling, Sql, Functions

Description: Learn how to use the IFNULL() function in MySQL to replace NULL values with a specified fallback value in queries and expressions.

---

## Introduction

`IFNULL()` is a MySQL function that returns the first argument if it is not NULL, or the second argument as a fallback if the first is NULL. It is one of the most commonly used NULL-handling functions and simplifies queries that need to substitute default values for missing data.

## Syntax

```sql
IFNULL(expression, fallback_value)
```

- If `expression` is NOT NULL, it returns `expression`.
- If `expression` IS NULL, it returns `fallback_value`.

## Basic Example

```sql
SELECT IFNULL(NULL, 'default');
-- Returns: 'default'

SELECT IFNULL('hello', 'default');
-- Returns: 'hello'

SELECT IFNULL(0, 'default');
-- Returns: 0  (0 is not NULL)
```

## Replace NULL in a Column

```sql
SELECT name, IFNULL(phone, 'No phone') AS phone_number
FROM customers;
```

This displays "No phone" instead of NULL for customers without a phone number.

## IFNULL with Numeric Columns

```sql
SELECT product_name, IFNULL(discount, 0) AS discount
FROM products;
```

This treats NULL discounts as 0, useful for arithmetic.

## IFNULL in Arithmetic

```sql
SELECT
  product_name,
  price,
  IFNULL(discount, 0) AS discount,
  price - IFNULL(discount, 0) AS final_price
FROM products;
```

Without `IFNULL`, any row with NULL discount would make `final_price` NULL.

## IFNULL with Aggregate Functions

```sql
SELECT
  department,
  IFNULL(AVG(salary), 0) AS avg_salary
FROM employees
GROUP BY department;
```

## IFNULL in WHERE Clause

```sql
SELECT name, email
FROM users
WHERE IFNULL(status, 'inactive') = 'inactive';
```

This matches rows where status is either NULL or explicitly 'inactive'.

## IFNULL vs COALESCE

- `IFNULL(a, b)` - takes exactly two arguments.
- `COALESCE(a, b, c, ...)` - takes any number of arguments, returns the first non-NULL.

Use `COALESCE` when you have more than one fallback value. Use `IFNULL` for simple two-option NULL replacement.

```sql
-- IFNULL: two arguments
SELECT IFNULL(middle_name, '') AS middle_name FROM users;

-- COALESCE: multiple fallback options
SELECT COALESCE(mobile, home_phone, work_phone, 'No contact') AS contact
FROM customers;
```

## IFNULL vs IF

`IFNULL(x, y)` is equivalent to `IF(x IS NULL, y, x)` but more concise for NULL checks.

```sql
-- These are equivalent
SELECT IFNULL(score, 0) FROM results;
SELECT IF(score IS NULL, 0, score) FROM results;
```

## IFNULL in UPDATE

```sql
UPDATE employees
SET bonus = IFNULL(bonus, 0) + 500
WHERE department = 'Sales';
```

This safely adds 500 to the bonus, treating NULL bonus as 0.

## IFNULL in INSERT

```sql
INSERT INTO orders (customer_id, notes)
VALUES (42, IFNULL(@notes_variable, 'No notes'));
```

## Performance Note

`IFNULL()` is evaluated at query execution time for each row. On large tables, indexing the underlying column is more impactful than the function itself. Avoid using `IFNULL` on indexed columns in `WHERE` clauses, as it can prevent index usage.

```sql
-- May not use index on `status`
WHERE IFNULL(status, 'inactive') = 'inactive'

-- Better: split the condition
WHERE status = 'inactive' OR status IS NULL
```

## Summary

`IFNULL()` is a simple and effective function for handling NULL values in MySQL. It replaces NULL with a default value, making queries safer for arithmetic and comparisons. For more than one fallback, use `COALESCE()` instead.
