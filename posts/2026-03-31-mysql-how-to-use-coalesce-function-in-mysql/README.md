# How to Use COALESCE() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, COALESCE, NULL Handling, SQL Functions

Description: Learn how to use the COALESCE() function in MySQL to return the first non-NULL value from a list of expressions with practical examples.

---

## What is COALESCE()

MySQL's `COALESCE()` function accepts two or more arguments and returns the first argument that is not NULL. If all arguments are NULL, it returns NULL.

Syntax:

```sql
COALESCE(value1, value2, value3, ...)
```

`COALESCE()` is standard SQL (ANSI) and is more versatile than `IFNULL()`, which only accepts two arguments.

## Basic Usage

```sql
SELECT COALESCE(NULL, NULL, 'third', 'fourth');
-- Output: third

SELECT COALESCE(NULL, 'second');
-- Output: second

SELECT COALESCE('first', 'second');
-- Output: first

SELECT COALESCE(NULL, NULL, NULL);
-- Output: NULL
```

## Fallback Chain for Contact Information

A classic use case is returning the best available contact method:

```sql
SELECT
  name,
  COALESCE(mobile_phone, home_phone, work_phone, 'No phone on file') AS contact_phone
FROM contacts;
```

This returns the mobile phone if available, then home phone, then work phone, and finally a default string.

## Using COALESCE() in SELECT

```sql
SELECT
  e.name,
  COALESCE(e.preferred_name, e.first_name, 'Employee') AS display_name,
  COALESCE(e.manager_name, 'No Manager') AS manager
FROM employees e;
```

## COALESCE() in Arithmetic

Prevents NULL propagation in calculations:

```sql
-- salary + bonus + allowance could be NULL if any component is NULL
SELECT
  name,
  salary + COALESCE(bonus, 0) + COALESCE(allowance, 0) AS total_comp
FROM employees;
```

## COALESCE() with Aggregate Functions

```sql
SELECT
  department,
  COALESCE(AVG(salary), 0) AS avg_salary,
  COALESCE(SUM(bonus), 0) AS total_bonus
FROM employees
GROUP BY department;
```

## COALESCE() vs IFNULL()

`IFNULL()` is limited to two arguments; `COALESCE()` accepts any number:

```sql
-- IFNULL: two values
SELECT IFNULL(phone, 'N/A') FROM contacts;

-- COALESCE: any number of fallbacks
SELECT COALESCE(mobile, home, work, 'N/A') FROM contacts;
```

## COALESCE() in WHERE Clauses

```sql
-- Find rows where at least one phone field is filled in
SELECT name
FROM contacts
WHERE COALESCE(mobile_phone, home_phone, work_phone) IS NOT NULL;
```

## COALESCE() with Subqueries

```sql
SELECT
  p.product_name,
  COALESCE(
    (SELECT sale_price FROM sales WHERE product_id = p.id LIMIT 1),
    p.regular_price
  ) AS effective_price
FROM products p;
```

## COALESCE() in UPDATE

```sql
-- Only update name if the new value is non-NULL
UPDATE employees
SET display_name = COALESCE(NEW_NAME_PARAMETER, display_name);
```

In practice with a stored procedure:

```sql
DELIMITER //
CREATE PROCEDURE update_profile(IN p_id INT, IN p_name VARCHAR(100))
BEGIN
  UPDATE employees
  SET name = COALESCE(p_name, name)
  WHERE id = p_id;
END //
DELIMITER ;
```

## COALESCE() and NULL Comparison

```sql
-- COALESCE also works with expressions
SELECT COALESCE(salary * 1.1, base_salary, 50000) AS adjusted_salary
FROM compensation;
```

## Summary

`COALESCE()` returns the first non-NULL value from a list of expressions, making it ideal for fallback chains across multiple optional columns, preventing NULL propagation in arithmetic, and providing default values. It is preferred over nested `IFNULL()` calls when working with more than two alternatives. All MySQL versions support `COALESCE()` as it is standard SQL.
