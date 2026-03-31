# What Is ONLY_FULL_GROUP_BY Mode in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL Mode, ONLY_FULL_GROUP_BY, GROUP BY, SQL Standard

Description: ONLY_FULL_GROUP_BY is a MySQL SQL mode that enforces SQL standard GROUP BY semantics, requiring all non-aggregated SELECT columns to appear in the GROUP BY clause.

---

## Overview

`ONLY_FULL_GROUP_BY` is a SQL mode flag that makes MySQL enforce standard SQL GROUP BY semantics. When enabled, any column in the `SELECT` list, `HAVING` clause, or `ORDER BY` clause that is not wrapped in an aggregate function (`SUM`, `COUNT`, `MAX`, etc.) must also appear in the `GROUP BY` clause. This prevents non-deterministic results where MySQL would otherwise silently pick an arbitrary value from the group.

This mode is enabled by default in MySQL 5.7.5 and later.

## The Problem Without ONLY_FULL_GROUP_BY

Without this mode, MySQL allows selecting non-grouped columns, which may return unpredictable values:

```sql
SET SESSION sql_mode = '';

-- Which 'name' from the group does MySQL return? Non-deterministic!
SELECT department_id, name, SUM(salary)
FROM employees
GROUP BY department_id;
```

MySQL returns a value for `name`, but which employee's name is returned from each group is undefined. This is a bug in the query logic even though it executes successfully.

## With ONLY_FULL_GROUP_BY Enabled

```sql
SET SESSION sql_mode = 'ONLY_FULL_GROUP_BY';

SELECT department_id, name, SUM(salary)
FROM employees
GROUP BY department_id;
-- ERROR 1055 (42000): 'name' is not in GROUP BY clause and contains nonaggregated column
```

## Fixing Queries for ONLY_FULL_GROUP_BY

**Option 1**: Add the column to GROUP BY:

```sql
SELECT department_id, name, SUM(salary)
FROM employees
GROUP BY department_id, name;
```

**Option 2**: Use an aggregate function:

```sql
SELECT department_id, MAX(name) AS sample_name, SUM(salary)
FROM employees
GROUP BY department_id;
```

**Option 3**: Use `ANY_VALUE()` to explicitly allow non-deterministic behavior:

```sql
SELECT department_id, ANY_VALUE(name) AS any_name, SUM(salary)
FROM employees
GROUP BY department_id;
```

`ANY_VALUE()` suppresses the error by telling MySQL you intentionally want an arbitrary value.

## Functional Dependency Exception

MySQL is smart about functional dependencies. If a column is functionally dependent on a grouped column, no error is raised:

```sql
-- If employee_id is the primary key, name is functionally dependent on it
SELECT employee_id, name, SUM(salary)
FROM employees
GROUP BY employee_id;  -- No error because name depends on employee_id
```

This is compliant with the SQL standard's functional dependency rules.

## ORDER BY and HAVING

`ONLY_FULL_GROUP_BY` also applies to `HAVING` and `ORDER BY`:

```sql
-- ERROR: status is not in GROUP BY or an aggregate
SELECT department_id, COUNT(*) FROM employees
GROUP BY department_id
HAVING status = 'active';

-- Fix:
SELECT department_id, COUNT(*) FROM employees
WHERE status = 'active'
GROUP BY department_id;
```

## Checking Whether the Mode Is Active

```sql
SELECT @@sql_mode LIKE '%ONLY_FULL_GROUP_BY%';
-- Returns 1 if active
```

## Summary

`ONLY_FULL_GROUP_BY` enforces standard SQL semantics for `GROUP BY` queries by requiring that all non-aggregate SELECT, HAVING, and ORDER BY columns appear in the GROUP BY clause. It prevents non-deterministic results from queries that silently pick arbitrary values from a group. `ANY_VALUE()` provides an escape hatch for cases where you intentionally want MySQL to pick any value. This mode is on by default in MySQL 5.7+ and should remain enabled in production.
