# How to Use ANY with Subqueries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Any Operator, Subquery, SQL, Database

Description: Learn how to use the ANY operator with subqueries in MySQL to compare a value against at least one row returned by a subquery, with practical examples.

---

## Introduction

The `ANY` operator in MySQL compares a value to the values returned by a subquery and returns TRUE if the comparison is true for at least one of them. It works with comparison operators like `>`, `<`, `>=`, `<=`, `=`, and `!=`. `SOME` is an exact synonym for `ANY` in MySQL.

## Basic Syntax

```sql
value comparison_operator ANY (subquery)
```

Returns TRUE if the comparison holds for at least one value in the subquery's result.

## ANY with > (Greater Than Any)

Find employees who earn more than at least one employee in the "Junior" grade:

```sql
SELECT name, salary
FROM employees
WHERE salary > ANY (
  SELECT salary FROM employees WHERE grade = 'Junior'
);
```

This is equivalent to:

```sql
-- Equivalent using MIN
SELECT name, salary FROM employees
WHERE salary > (SELECT MIN(salary) FROM employees WHERE grade = 'Junior');
```

## ANY with < (Less Than Any)

Find products cheaper than at least one premium product:

```sql
SELECT product_name, price
FROM products
WHERE price < ANY (
  SELECT price FROM products WHERE tier = 'premium'
);
```

Returns products priced below the most expensive premium product (i.e., below MAX).

## = ANY is Equivalent to IN

```sql
-- Using = ANY
SELECT id, name FROM customers
WHERE status = ANY (SELECT status FROM active_statuses);

-- Equivalent with IN
SELECT id, name FROM customers
WHERE status IN (SELECT status FROM active_statuses);
```

Use `IN` for readability. `= ANY` is the formal SQL standard equivalent.

## ANY with >= and <=

```sql
-- Find orders with amount >= at least one threshold value
SELECT order_id, amount
FROM orders
WHERE amount >= ANY (
  SELECT threshold FROM discount_tiers
);
```

## != ANY

`!= ANY` means "not equal to at least one value" - this is almost always TRUE unless the subquery returns exactly one value and it matches.

```sql
-- TRUE for almost all rows (not very useful)
SELECT name FROM employees
WHERE department != ANY (SELECT department FROM departments);
```

This is quite different from `!= ALL` (which means NOT IN).

## Practical Example: Comparing to Department Averages

Find employees earning more than the average salary of at least one department:

```sql
SELECT DISTINCT e.name, e.salary
FROM employees e
WHERE e.salary > ANY (
  SELECT AVG(salary)
  FROM employees
  GROUP BY department
);
```

## ANY in a Real-World Scenario

Find products that are more expensive than at least one competitor's product in the same category:

```sql
SELECT p.name, p.price, p.category
FROM our_products p
WHERE p.price > ANY (
  SELECT cp.price
  FROM competitor_products cp
  WHERE cp.category = p.category
);
```

## Edge Cases

### Empty Subquery

If the subquery returns no rows, `ANY` evaluates to FALSE:

```sql
-- If no rows match the subquery, this returns nothing
SELECT name FROM employees
WHERE salary > ANY (
  SELECT salary FROM employees WHERE grade = 'nonexistent'
);
-- Returns empty set
```

### NULL in Subquery

If the subquery contains NULLs, comparisons involving NULL return UNKNOWN, and those rows may be excluded. Filter NULLs from the subquery to avoid surprises:

```sql
SELECT name FROM employees
WHERE salary > ANY (
  SELECT salary FROM bonus_recipients WHERE salary IS NOT NULL
);
```

## ANY vs ALL Quick Reference

| Operator | TRUE when | Equivalent to |
|----------|-----------|---------------|
| `> ANY` | Greater than at least one value | `> MIN(...)` |
| `< ANY` | Less than at least one value | `< MAX(...)` |
| `= ANY` | Equal to at least one value | `IN (...)` |
| `> ALL` | Greater than every value | `> MAX(...)` |
| `< ALL` | Less than every value | `< MIN(...)` |
| `!= ALL` | Not equal to any value | `NOT IN (...)` |

## SOME is a Synonym for ANY

MySQL supports `SOME` as an alias for `ANY`. Both are part of the SQL standard.

```sql
-- These are equivalent
WHERE salary > ANY (SELECT salary FROM juniors)
WHERE salary > SOME (SELECT salary FROM juniors)
```

## Summary

`ANY` (or `SOME`) returns TRUE when a comparison holds for at least one value in a subquery result. It is equivalent to `> MIN()` when using `>`, `< MAX()` when using `<`, and `IN` when using `=`. The subquery must return at least one non-NULL row for `ANY` to evaluate to TRUE. Use `IN` as a readable alternative to `= ANY`.
