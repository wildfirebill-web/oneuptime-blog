# How to Use ANY and ALL Operators in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Any, All, Subquery, Sql

Description: Learn how to use the ANY and ALL operators in MySQL to compare a value against a set of values returned by a subquery.

---

## Introduction

`ANY` and `ALL` are comparison operators in MySQL used with subqueries to compare a value against multiple results. `ANY` returns TRUE if the comparison is true for at least one value in the subquery result. `ALL` returns TRUE only if the comparison is true for every value in the subquery result.

## ANY Operator Syntax

```sql
value comparison_operator ANY (subquery)
```

Returns TRUE if the comparison holds for at least one value returned by the subquery. `SOME` is a synonym for `ANY`.

## ALL Operator Syntax

```sql
value comparison_operator ALL (subquery)
```

Returns TRUE only if the comparison holds for every value returned by the subquery.

## Simple ANY Example

Find employees who earn more than at least one employee in the Marketing department:

```sql
SELECT name, salary
FROM employees
WHERE salary > ANY (
  SELECT salary FROM employees WHERE department = 'Marketing'
);
```

This returns employees whose salary is greater than the lowest salary in Marketing.

## Simple ALL Example

Find employees who earn more than ALL employees in the Marketing department:

```sql
SELECT name, salary
FROM employees
WHERE salary > ALL (
  SELECT salary FROM employees WHERE department = 'Marketing'
);
```

This returns employees whose salary is greater than the highest salary in Marketing.

## ANY Equivalent to > MIN

```sql
-- These are equivalent
WHERE salary > ANY (SELECT salary FROM employees WHERE department = 'Marketing')
WHERE salary > (SELECT MIN(salary) FROM employees WHERE department = 'Marketing')
```

## ALL Equivalent to > MAX

```sql
-- These are equivalent
WHERE salary > ALL (SELECT salary FROM employees WHERE department = 'Marketing')
WHERE salary > (SELECT MAX(salary) FROM employees WHERE department = 'Marketing')
```

## = ANY is Equivalent to IN

```sql
-- These are equivalent
WHERE department = ANY (SELECT department FROM headquarters)
WHERE department IN (SELECT department FROM headquarters)
```

## != ALL is Equivalent to NOT IN

```sql
-- These are equivalent
WHERE department != ALL (SELECT department FROM blacklist)
WHERE department NOT IN (SELECT department FROM blacklist)
```

## ANY with Different Comparison Operators

```sql
-- Greater than or equal to any value
SELECT product_name, price
FROM products
WHERE price >= ANY (SELECT price FROM competitor_products WHERE category = 'phones');

-- Less than any value
SELECT name, score
FROM students
WHERE score < ANY (SELECT passing_score FROM grade_thresholds);
```

## ALL with Different Comparison Operators

```sql
-- Less than all values in the subquery
SELECT name, price
FROM products
WHERE price < ALL (
  SELECT price FROM products WHERE category = 'premium'
);

-- Equal to all values (rarely useful but valid)
SELECT config_key
FROM server_configs
WHERE config_value = ALL (
  SELECT default_value FROM config_defaults WHERE server_configs.config_key = config_key
);
```

## Behavior with Empty Subquery

- `value op ANY (empty)` returns FALSE.
- `value op ALL (empty)` returns TRUE (vacuous truth).

```sql
-- If no rows match the subquery
WHERE salary > ANY (SELECT salary FROM employees WHERE department = 'NonExistent')
-- Returns: FALSE (no values to compare against)

WHERE salary > ALL (SELECT salary FROM employees WHERE department = 'NonExistent')
-- Returns: TRUE (all zero values are beaten)
```

## Behavior with NULL in Subquery

If the subquery contains NULL values, comparisons can return NULL (unknown), leading to rows being excluded from the result.

```sql
-- NULL in subquery can cause unexpected results
WHERE salary > ALL (SELECT salary FROM employees)
-- If any salary is NULL, this may return no rows
```

## Practical Example: Price Comparison

Find products priced higher than all products in the "Budget" category:

```sql
SELECT p.name, p.price
FROM products p
WHERE p.price > ALL (
  SELECT price FROM products WHERE category = 'Budget'
)
AND p.category != 'Budget';
```

## Practical Example: Conditional Filtering

Find employees whose salary is above average in at least one department:

```sql
SELECT DISTINCT e.name, e.salary
FROM employees e
WHERE e.salary > ANY (
  SELECT AVG(salary)
  FROM employees
  GROUP BY department
);
```

## Summary

`ANY` and `ALL` operators compare a value against a set of values from a subquery. `ANY` (or `SOME`) returns true if any comparison is satisfied; `ALL` returns true only if every comparison is satisfied. They are equivalent to `> MIN`, `> MAX`, `IN`, and `NOT IN` in many cases, but provide a more expressive way to write certain comparisons.
