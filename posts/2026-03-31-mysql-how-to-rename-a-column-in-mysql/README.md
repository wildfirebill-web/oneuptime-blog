# How to Rename a Column in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Ddl, Alter Table, Schema Design

Description: Learn how to rename a column in MySQL using RENAME COLUMN (MySQL 8.0) and the CHANGE clause for older versions, with tips on handling dependencies.

---

## Two Ways to Rename a Column

MySQL offers two approaches depending on the version:

1. `RENAME COLUMN` - MySQL 8.0+ (preferred, concise)
2. `CHANGE` - works in all versions, but requires re-specifying the full column definition

## Using RENAME COLUMN (MySQL 8.0+)

```sql
ALTER TABLE table_name
RENAME COLUMN old_name TO new_name;
```

This is the simplest and safest approach. It only changes the column name without altering its data type or constraints.

```sql
-- Example
ALTER TABLE users
RENAME COLUMN usr_email TO email;
```

## Renaming Multiple Columns

```sql
ALTER TABLE users
RENAME COLUMN usr_email    TO email,
RENAME COLUMN usr_username TO username,
RENAME COLUMN usr_phone    TO phone;
```

## Using CHANGE (MySQL 5.7 and Below)

`CHANGE` renames the column but also requires you to re-specify the column definition:

```sql
ALTER TABLE table_name
CHANGE old_column_name new_column_name column_definition;
```

```sql
-- Example: rename and keep the same definition
ALTER TABLE users
CHANGE usr_email email VARCHAR(100) NOT NULL;
```

If the column definition is wrong or incomplete, it will alter the column's type or constraints, which can be destructive. Always check `DESCRIBE table_name` first.

## Using MODIFY vs CHANGE vs RENAME COLUMN

```text
RENAME COLUMN  - MySQL 8.0+ - rename only, no definition change
CHANGE         - all versions - rename + can modify definition
MODIFY         - all versions - modify definition only, cannot rename
```

## Checking the Column Definition Before Renaming

```sql
SHOW COLUMNS FROM users LIKE 'usr_email';
```

Output:

```text
Field     | Type         | Null | Key | Default | Extra
----------+--------------+------+-----+---------+-------
usr_email | varchar(100) | NO   |     | NULL    |
```

Use this output to build the correct `CHANGE` statement if on MySQL 5.7.

## Impact on Indexes and Foreign Keys

Renaming a column with `RENAME COLUMN` automatically updates all indexes that reference it. However, application code (queries, ORMs, stored procedures, views) that reference the old column name must be updated manually.

Check for views and stored procedures that use the old column name:

```sql
SELECT
    ROUTINE_TYPE,
    ROUTINE_NAME,
    ROUTINE_DEFINITION
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = DATABASE()
  AND ROUTINE_DEFINITION LIKE '%usr_email%';

SELECT TABLE_NAME, VIEW_DEFINITION
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND VIEW_DEFINITION LIKE '%usr_email%';
```

## Online DDL Considerations

In MySQL 8.0 with InnoDB, `RENAME COLUMN` is an instant operation (no table rebuild needed):

```sql
ALTER TABLE users
RENAME COLUMN usr_email TO email,
ALGORITHM=INSTANT;
```

`ALGORITHM=INSTANT` makes the rename metadata-only, completing in milliseconds regardless of table size.

## Summary

Use `ALTER TABLE ... RENAME COLUMN old TO new` in MySQL 8.0+ for a safe, concise column rename that does not alter the column's data type or constraints. On MySQL 5.7 and earlier, use the `CHANGE` clause with the full column definition. Both approaches automatically update index references, but application queries, views, and stored procedures must be updated manually to use the new name.
