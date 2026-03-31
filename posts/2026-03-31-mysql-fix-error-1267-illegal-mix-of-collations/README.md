# How to Fix ERROR 1267 Illegal Mix of Collations in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Collation, Error, Charset, String

Description: Fix MySQL ERROR 1267 Illegal mix of collations by aligning column, table, and database character sets and using COLLATE or CONVERT in queries.

---

MySQL ERROR 1267 occurs when a query compares or combines string values that use incompatible collations. The error message reads: `ERROR 1267 (HY000): Illegal mix of collations (utf8mb4_unicode_ci,IMPLICIT) and (utf8mb4_general_ci,IMPLICIT) for operation '='`.

## Understand Collation Precedence

MySQL resolves collation conflicts using a coercibility hierarchy. When two values have different collations at the same coercibility level, MySQL cannot decide which to use and throws the error.

## Diagnose the Problem

Identify which columns have mismatched collations:

```sql
-- Check database default collation
SELECT DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = DATABASE();

-- Check table and column collations
SELECT TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_TYPE IN ('char', 'varchar', 'text', 'tinytext', 'mediumtext', 'longtext');
```

## Fix: Align Column Collations

The most permanent fix is to make all columns use the same collation:

```sql
-- Change a single column
ALTER TABLE users
  MODIFY COLUMN username VARCHAR(100)
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Change all string columns in a table
ALTER TABLE users
  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Fix: Use COLLATE in the Query

For a quick fix without altering tables, specify the collation in the query:

```sql
-- Force both sides to use the same collation
SELECT * FROM users u
JOIN sessions s ON u.username COLLATE utf8mb4_unicode_ci = s.username;

-- Or use CONVERT
SELECT * FROM users u
JOIN sessions s
  ON CONVERT(u.username USING utf8mb4) = CONVERT(s.username USING utf8mb4);
```

## Fix: Standardize the Database Collation

Set a default collation at the database level so new tables inherit it:

```sql
ALTER DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

New tables created without an explicit collation will use this default.

## Fix: Session-Level Collation for Connections

Application frameworks sometimes send connection collations that differ from the server default. Set them explicitly in your connection string:

```text
jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=UTF-8&connectionCollation=utf8mb4_unicode_ci
```

Or in Python:

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    user='app',
    password='secret',
    database='mydb',
    charset='utf8mb4',
    collation='utf8mb4_unicode_ci'
)
```

## Fix: Update Stored Procedures and Views

Collation issues in stored routines require dropping and recreating them:

```sql
SHOW CREATE PROCEDURE my_proc\G
-- Note the COLLATE used
DROP PROCEDURE my_proc;
-- Recreate with consistent collation
```

## Summary

ERROR 1267 is a collation mismatch between compared strings. The best long-term fix is standardizing on `utf8mb4_unicode_ci` across your database, tables, and columns using `ALTER DATABASE` and `ALTER TABLE ... CONVERT TO`. For quick fixes, use `COLLATE` or `CONVERT` in your queries to force a consistent collation at runtime.
