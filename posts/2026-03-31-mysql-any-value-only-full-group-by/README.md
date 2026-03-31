# How to Use ANY_VALUE() Function in MySQL with ONLY_FULL_GROUP_BY

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Aggregate Function, GROUP BY

Description: Learn how MySQL's ANY_VALUE() function suppresses ONLY_FULL_GROUP_BY errors by explicitly marking a column as non-deterministic in GROUP BY queries.

---

## The ONLY_FULL_GROUP_BY Problem

MySQL's `ONLY_FULL_GROUP_BY` SQL mode (enabled by default since MySQL 5.7.5) requires that every column in a `SELECT` list either:
1. Appears in the `GROUP BY` clause, or
2. Is wrapped in an aggregate function (`MAX()`, `MIN()`, `COUNT()`, etc.)

If a query violates this rule, MySQL raises an error:

```sql
SELECT
  department_id,
  name,          -- ERROR: not in GROUP BY and not aggregated
  MAX(salary)
FROM employees
GROUP BY department_id;
-- ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP BY clause
```

## What Is ANY_VALUE()?

`ANY_VALUE()` is a special aggregate function introduced in MySQL 5.7.5. It returns an arbitrary value from the group for the given expression and suppresses the `ONLY_FULL_GROUP_BY` error. It is an explicit signal to MySQL: "I know this is non-deterministic and I accept that."

```sql
SELECT
  department_id,
  ANY_VALUE(name) AS sample_name,  -- explicitly non-deterministic
  MAX(salary) AS max_salary
FROM employees
GROUP BY department_id;
```

## When to Use ANY_VALUE()

Use `ANY_VALUE()` when:
- The column is functionally dependent on the `GROUP BY` key but MySQL cannot detect this
- You genuinely don't care which value is returned (e.g., picking any representative value)
- You're migrating code from MySQL 5.6 where `ONLY_FULL_GROUP_BY` was not enforced

## Functional Dependency Example

If `employee_id` is the primary key and you group by it, other columns are functionally dependent:

```sql
-- MySQL may not detect this functional dependency
SELECT
  employee_id,
  ANY_VALUE(name) AS name,  -- safe: employee_id is PK, name is unique per group
  ANY_VALUE(email) AS email
FROM employees
GROUP BY employee_id;
```

A more correct alternative is to simply add `name` and `email` to the `GROUP BY`:

```sql
SELECT employee_id, name, email
FROM employees
GROUP BY employee_id, name, email;
```

## Practical Example: Latest Entry per Group

A common pattern is to get the most recent row per group:

```sql
-- Get the most recent order per customer
SELECT
  customer_id,
  ANY_VALUE(order_date) AS latest_date,
  MAX(order_date) AS confirmed_max_date,
  ANY_VALUE(order_total) AS any_total
FROM orders
GROUP BY customer_id
HAVING MAX(order_date) = ANY_VALUE(order_date);  -- not reliable
```

For deterministic results, a subquery or window function is better:

```sql
SELECT customer_id, order_date, order_total
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
  FROM orders
) t
WHERE rn = 1;
```

## ANY_VALUE() vs Disabling ONLY_FULL_GROUP_BY

Do not disable `ONLY_FULL_GROUP_BY` globally as a workaround:

```sql
-- Avoid this approach in production
SET sql_mode = REPLACE(@@sql_mode, 'ONLY_FULL_GROUP_BY', '');
```

`ANY_VALUE()` is the correct targeted approach - it makes the non-determinism explicit and reviewable in the query.

## Summary

`ANY_VALUE()` suppresses `ONLY_FULL_GROUP_BY` errors by explicitly marking a column as non-deterministic within a group. Use it when you intentionally want an arbitrary value from the group or when MySQL cannot infer functional dependency. For deterministic results, prefer `MAX()`, `MIN()`, or window functions instead.
