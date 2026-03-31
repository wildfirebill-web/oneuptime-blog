# How to Use SOME with Subqueries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SOME, Subquery, ANY, Comparison

Description: Learn how SOME works as an alias for ANY in MySQL subqueries and when to use it for clear, readable conditional comparisons.

---

## What Is SOME in MySQL

`SOME` is a synonym for `ANY` in MySQL. Both keywords have identical behavior. The SQL standard defines both, and MySQL supports them interchangeably. `SOME` was introduced to provide a more natural-language reading in certain conditions.

```sql
value comparison_operator SOME (subquery)
-- is exactly equivalent to:
value comparison_operator ANY (subquery)
```

## Basic Usage

Find employees who earn more than at least one employee in the 'Intern' grade:

```sql
SELECT id, name, salary
FROM employees
WHERE salary > SOME (
  SELECT salary FROM employees WHERE grade = 'Intern'
);
```

The same query using `ANY`:

```sql
SELECT id, name, salary
FROM employees
WHERE salary > ANY (
  SELECT salary FROM employees WHERE grade = 'Intern'
);
```

Both produce identical results.

## When SOME Is More Readable

The natural-language reading of `> SOME` is "greater than some" - which maps naturally to "greater than at least one". Some developers find `SOME` more expressive in certain contexts:

```sql
-- "Find products cheaper than some premium product"
SELECT id, name, price
FROM products
WHERE price < SOME (
  SELECT price FROM products WHERE category = 'Premium'
);
```

## = SOME Is Equivalent to IN

Like `ANY`, `= SOME` is equivalent to the `IN` operator:

```sql
-- Using = SOME
SELECT id, title
FROM articles
WHERE author_id = SOME (
  SELECT id FROM editors WHERE active = 1
);

-- Equivalent using IN
SELECT id, title
FROM articles
WHERE author_id IN (
  SELECT id FROM editors WHERE active = 1
);
```

For equality checks, `IN` is typically preferred since most developers are more familiar with it.

## Practical Example with Correlated Subquery

Find departments where at least one employee has a salary above 100,000:

```sql
SELECT DISTINCT department_id
FROM employees e
WHERE 100000 < SOME (
  SELECT salary
  FROM employees
  WHERE department_id = e.department_id
);
```

## SOME vs ANY vs IN - Quick Reference

| Use Case                            | Recommended  |
|-------------------------------------|-------------|
| Equality match against a list       | IN           |
| "Greater than at least one" type    | ANY or SOME  |
| Readability preference              | SOME for naturalness |
| Maximum comparison without MAX()    | > ANY / > SOME |

## Verifying Execution with EXPLAIN

```sql
EXPLAIN
SELECT id, name
FROM products
WHERE price < SOME (
  SELECT price FROM products WHERE category = 'Premium'
);
```

The query plan for `SOME` will be identical to the equivalent `ANY` query.

## Summary

`SOME` is a standard SQL synonym for `ANY` in MySQL and behaves identically. Use `SOME` when it makes the query intent more readable in natural language. For equality comparisons, `IN` remains the most idiomatic choice. Choose whichever form improves clarity for your team.
