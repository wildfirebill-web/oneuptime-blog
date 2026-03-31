# How to Use CHARSET() and COLLATION() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Function, Collation, Character Set, String

Description: Learn how to use CHARSET() and COLLATION() functions in MySQL to inspect the character set and collation of string expressions.

---

## Overview

MySQL provides two introspection functions - `CHARSET()` and `COLLATION()` - that let you determine the character set and collation of any string expression at runtime. These are particularly useful when debugging character set mismatches or verifying that data is handled correctly.

## CHARSET() Function

The `CHARSET()` function returns the character set of a string expression. It takes a single argument and returns the name of the character set as a string.

```sql
SELECT CHARSET('hello');
-- Returns: utf8mb4

SELECT CHARSET(first_name) FROM users LIMIT 1;
-- Returns the charset of the column
```

You can also inspect the character set of a column directly:

```sql
SELECT CHARSET(description) AS col_charset
FROM products
LIMIT 1;
```

The function is especially useful when comparing character sets across columns or checking the result of string functions:

```sql
SELECT
  CHARSET(name)              AS name_charset,
  CHARSET(CONVERT(name USING latin1)) AS converted_charset
FROM customers
LIMIT 1;
```

## COLLATION() Function

The `COLLATION()` function returns the collation of a string expression. Collation determines how string comparisons are performed - whether they are case-sensitive, accent-sensitive, and so on.

```sql
SELECT COLLATION('hello');
-- Returns: utf8mb4_0900_ai_ci

SELECT COLLATION(email) FROM users LIMIT 1;
```

You can pair `CHARSET()` and `COLLATION()` together to get a full picture of how a string is handled:

```sql
SELECT
  CHARSET(username)    AS charset,
  COLLATION(username)  AS collation
FROM users
LIMIT 1;
```

## Checking System Variables

These functions respect the current `character_set_connection` and `collation_connection` session variables, so they are useful for verifying connection-level settings:

```sql
SHOW VARIABLES LIKE 'character_set_connection';
SHOW VARIABLES LIKE 'collation_connection';

SELECT CHARSET('test'), COLLATION('test');
```

## Using CHARSET() and COLLATION() in Diagnostics

A common use case is diagnosing collation mismatch errors. When you join two columns that have different collations, MySQL may throw `Illegal mix of collations`. These functions help you identify the problem:

```sql
SELECT
  CHARSET(a.name)     AS a_charset,
  COLLATION(a.name)   AS a_collation,
  CHARSET(b.name)     AS b_charset,
  COLLATION(b.name)   AS b_collation
FROM table_a a
JOIN table_b b ON a.id = b.id
LIMIT 1;
```

Once identified, you can coerce one side:

```sql
SELECT *
FROM table_a a
JOIN table_b b
  ON a.name = CONVERT(b.name USING utf8mb4) COLLATE utf8mb4_0900_ai_ci;
```

## Practical Example: Auditing Column Charsets

You can use these functions together with `INFORMATION_SCHEMA` to audit all columns in a database:

```sql
SELECT
  TABLE_NAME,
  COLUMN_NAME,
  CHARACTER_SET_NAME,
  COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'mydb'
  AND CHARACTER_SET_NAME IS NOT NULL
ORDER BY TABLE_NAME, COLUMN_NAME;
```

## Summary

`CHARSET()` and `COLLATION()` are lightweight introspection functions that return the character set and collation of any string expression. They are invaluable for debugging character set mismatch errors, verifying connection settings, and ensuring consistent string handling across tables and queries.
