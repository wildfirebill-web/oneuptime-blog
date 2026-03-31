# What Is ONLY_FULL_GROUP_BY Mode in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL Mode, GROUP BY, Query Optimization

Description: ONLY_FULL_GROUP_BY is a MySQL SQL mode that requires all non-aggregated SELECT columns to appear in the GROUP BY clause, enforcing SQL standard compliance.

---

## Overview

`ONLY_FULL_GROUP_BY` is a SQL mode flag in MySQL that enforces stricter GROUP BY behavior aligned with the SQL standard. When active, every column in the SELECT list that is not part of an aggregate function must either appear in the GROUP BY clause or be functionally dependent on a GROUP BY column.

Without this mode, MySQL allows selecting non-grouped columns, which produces non-deterministic (unpredictable) results.

## Checking If the Mode Is Active

```sql
SELECT @@sql_mode LIKE '%ONLY_FULL_GROUP_BY%';

SHOW VARIABLES LIKE 'sql_mode';
```

## The Problem It Solves

```sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer_id INT,
  amount DECIMAL(10,2),
  status VARCHAR(20)
);

-- Without ONLY_FULL_GROUP_BY: runs but 'status' is non-deterministic
SELECT customer_id, status, SUM(amount)
FROM orders
GROUP BY customer_id;

-- With ONLY_FULL_GROUP_BY: error
-- ERROR 1055 (42000): 'mydb.orders.status' isn't in GROUP BY
```

## Correct Query Patterns

### Include All Non-Aggregated Columns

```sql
-- Correct: status is included in GROUP BY
SELECT customer_id, status, SUM(amount)
FROM orders
GROUP BY customer_id, status;
```

### Use an Aggregate Function

```sql
-- Use MAX or MIN to pick a representative value
SELECT customer_id, MAX(status) AS latest_status, SUM(amount)
FROM orders
GROUP BY customer_id;
```

### Use ANY_VALUE()

```sql
-- Explicitly tell MySQL you accept any arbitrary value
SELECT customer_id, ANY_VALUE(status) AS sample_status, SUM(amount)
FROM orders
GROUP BY customer_id;
```

## Functional Dependency Exception

MySQL 8.0 is smart enough to detect functional dependencies. If a column is functionally determined by a GROUP BY column (like a primary key relationship), it is allowed:

```sql
CREATE TABLE employees (
  emp_id INT PRIMARY KEY,
  dept_id INT,
  name VARCHAR(100)
);

-- name is functionally dependent on emp_id (primary key)
-- This is allowed even with ONLY_FULL_GROUP_BY
SELECT emp_id, name, dept_id
FROM employees
GROUP BY emp_id;
```

## Disabling ONLY_FULL_GROUP_BY

While not recommended for production, you can disable it for legacy queries:

```sql
-- Remove from session
SET SESSION sql_mode = REPLACE(@@SESSION.sql_mode, 'ONLY_FULL_GROUP_BY', '');

-- Or set explicitly
SET SESSION sql_mode = 'STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION';
```

## Enabling ONLY_FULL_GROUP_BY

```sql
SET SESSION sql_mode = CONCAT(@@sql_mode, ',ONLY_FULL_GROUP_BY');
```

In `my.cnf`:

```text
[mysqld]
sql_mode = ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION
```

## Common Migration Patterns

When enabling this mode on an existing app, you may need to rewrite queries:

```sql
-- Old non-compliant query
SELECT user_id, email, COUNT(*) as order_count
FROM orders
GROUP BY user_id;

-- Fixed: aggregate email or add to GROUP BY
SELECT user_id, ANY_VALUE(email) AS email, COUNT(*) as order_count
FROM orders
GROUP BY user_id;
```

## Summary

`ONLY_FULL_GROUP_BY` enforces SQL standard compliance by requiring all non-aggregated SELECT columns to be in the GROUP BY clause or determined by it. It is part of MySQL 8.0's default SQL mode and prevents non-deterministic query results. Use `ANY_VALUE()`, aggregates, or proper GROUP BY clauses to write compliant queries.
