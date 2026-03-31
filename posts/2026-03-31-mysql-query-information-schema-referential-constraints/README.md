# How to Query INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Foreign Key, Referential Integrity, Schema

Description: Learn how to query INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS in MySQL to inspect foreign key cascade rules, update rules, and referential integrity configuration.

---

## Overview

`INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS` extends foreign key information by providing the `UPDATE_RULE` and `DELETE_RULE` for each foreign key constraint. These rules define what happens to child rows when parent rows are updated or deleted.

## Basic Query

```sql
SELECT
  CONSTRAINT_SCHEMA,
  CONSTRAINT_NAME,
  UNIQUE_CONSTRAINT_SCHEMA,
  UNIQUE_CONSTRAINT_NAME,
  MATCH_OPTION,
  UPDATE_RULE,
  DELETE_RULE
FROM information_schema.REFERENTIAL_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'myapp'
ORDER BY CONSTRAINT_NAME;
```

## Understanding CASCADE Rules

The `UPDATE_RULE` and `DELETE_RULE` columns can contain:

```text
RESTRICT   - Prevent update/delete if child rows exist
CASCADE    - Automatically update/delete child rows
SET NULL   - Set FK column to NULL in child rows
NO ACTION  - Like RESTRICT but deferred (effectively same in MySQL)
SET DEFAULT - Not supported by InnoDB (parser accepts it)
```

## Finding All Foreign Keys with Their Cascade Rules

```sql
SELECT
  rc.CONSTRAINT_NAME,
  kcu.TABLE_NAME AS child_table,
  kcu.COLUMN_NAME AS child_column,
  kcu.REFERENCED_TABLE_NAME AS parent_table,
  kcu.REFERENCED_COLUMN_NAME AS parent_column,
  rc.UPDATE_RULE,
  rc.DELETE_RULE
FROM information_schema.REFERENTIAL_CONSTRAINTS rc
JOIN information_schema.KEY_COLUMN_USAGE kcu
  ON rc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
  AND rc.CONSTRAINT_SCHEMA = kcu.TABLE_SCHEMA
WHERE rc.CONSTRAINT_SCHEMA = 'myapp'
ORDER BY child_table, rc.CONSTRAINT_NAME;
```

## Finding Dangerous CASCADE DELETE Relationships

Cascade deletes can accidentally remove large amounts of data:

```sql
SELECT
  rc.CONSTRAINT_NAME,
  kcu.TABLE_NAME AS child_table,
  kcu.REFERENCED_TABLE_NAME AS parent_table,
  rc.DELETE_RULE
FROM information_schema.REFERENTIAL_CONSTRAINTS rc
JOIN information_schema.KEY_COLUMN_USAGE kcu
  ON rc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
  AND rc.CONSTRAINT_SCHEMA = kcu.TABLE_SCHEMA
WHERE rc.CONSTRAINT_SCHEMA = 'myapp'
  AND rc.DELETE_RULE = 'CASCADE';
```

## Finding FK Without Cascade (RESTRICT/NO ACTION)

```sql
SELECT
  CONSTRAINT_NAME,
  UPDATE_RULE,
  DELETE_RULE
FROM information_schema.REFERENTIAL_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'myapp'
  AND DELETE_RULE IN ('RESTRICT', 'NO ACTION');
```

## Generating ALTER TABLE Statements to Change Cascade Rules

```sql
SELECT
  CONCAT(
    'ALTER TABLE `', kcu.TABLE_NAME, '` ',
    'DROP FOREIGN KEY `', rc.CONSTRAINT_NAME, '`; ',
    'ALTER TABLE `', kcu.TABLE_NAME, '` ',
    'ADD CONSTRAINT `', rc.CONSTRAINT_NAME, '` ',
    'FOREIGN KEY (`', kcu.COLUMN_NAME, '`) ',
    'REFERENCES `', kcu.REFERENCED_TABLE_NAME, '`(`', kcu.REFERENCED_COLUMN_NAME, '`) ',
    'ON DELETE CASCADE ON UPDATE CASCADE;'
  ) AS alter_statement
FROM information_schema.REFERENTIAL_CONSTRAINTS rc
JOIN information_schema.KEY_COLUMN_USAGE kcu
  ON rc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
  AND rc.CONSTRAINT_SCHEMA = kcu.TABLE_SCHEMA
WHERE rc.CONSTRAINT_SCHEMA = 'myapp'
  AND rc.DELETE_RULE = 'RESTRICT';
```

## Summary

`INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS` reveals the cascade behavior of every foreign key in your schema. Auditing `UPDATE_RULE` and `DELETE_RULE` helps identify accidental cascade deletes, missing cascade updates, and orphan-prevention issues. Join it with `KEY_COLUMN_USAGE` to get a complete picture of your referential integrity configuration.
