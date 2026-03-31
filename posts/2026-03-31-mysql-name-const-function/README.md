# How to Use NAME_CONST() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Function, Stored Procedure, Replication, Column

Description: Learn how MySQL's NAME_CONST() function works, when it is used in stored procedure replication, and how it affects column naming.

---

## Overview

`NAME_CONST(name, value)` is an internal MySQL function that produces a column with a specific name and a constant value. It is primarily used by MySQL's binary log replication mechanism when replicating stored procedures that use local variables. Understanding it helps when reading binary logs or debugging replication issues.

## Basic Syntax and Behavior

`NAME_CONST()` takes two arguments: a string name and a constant value. It returns the value, but assigns the given name as the column alias in a result set:

```sql
SELECT NAME_CONST('greeting', 'Hello, World!');
-- Column name: greeting
-- Value:       Hello, World!

SELECT NAME_CONST('answer', 42);
-- Column name: answer
-- Value:       42
```

Without `NAME_CONST()`, an expression like `SELECT 42` would produce a column named `42`. With it, you can control the output column name.

## How MySQL Uses NAME_CONST() in Replication

When MySQL replicates a stored procedure call using statement-based replication (SBR), it rewrites local variable references using `NAME_CONST()` so that the replica can interpret the statement correctly. For example, a stored procedure like:

```sql
CREATE PROCEDURE greet_user(IN username VARCHAR(100))
BEGIN
  SELECT CONCAT('Hello, ', username) AS greeting;
END;
```

When called as `CALL greet_user('Alice')` may appear in the binary log as:

```sql
SELECT CONCAT('Hello, ', NAME_CONST('username', 'Alice')) AS greeting;
```

This ensures the replica can execute the exact same query without needing the procedure definition.

## Viewing NAME_CONST() in Binary Logs

You can observe `NAME_CONST()` usage by reading binary logs with `mysqlbinlog`:

```bash
mysqlbinlog --base64-output=DECODE-ROWS --verbose /var/lib/mysql/binlog.000001 | grep NAME_CONST
```

## Practical Example: Column Naming

`NAME_CONST()` can be used in regular queries to force a specific column name for a constant value, which is occasionally useful in `UNION` queries or dynamic result sets:

```sql
SELECT NAME_CONST('status', 'active') AS status
UNION ALL
SELECT status FROM users WHERE active = 1;
```

## Limitations

`NAME_CONST()` has strict requirements:

```sql
-- The first argument must be a string literal
-- The second argument must be a constant (literal value)

-- This will fail:
SELECT NAME_CONST('col', NOW());  -- Error: not a constant

-- This works:
SELECT NAME_CONST('version', '8.0.32');
```

The second argument must be a literal constant - expressions and function calls are not allowed.

## Checking the Function in Action

```sql
-- You can see the column naming effect clearly
SELECT
  NAME_CONST('db_version', '8.0'),
  NAME_CONST('env', 'production');
-- Returns two columns named db_version and env
```

## Summary

`NAME_CONST(name, value)` is an internal MySQL function used primarily in statement-based replication to preserve variable names when stored procedures are replicated. While rarely called directly by developers, it can be used to assign custom column names to constant values in queries. Understanding it helps when analyzing binary logs or diagnosing replication-related query rewrites.
