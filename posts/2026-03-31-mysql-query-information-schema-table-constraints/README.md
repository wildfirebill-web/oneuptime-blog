# How to Query INFORMATION_SCHEMA.TABLE_CONSTRAINTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Constraint, Schema, Foreign Key

Description: Learn how to query INFORMATION_SCHEMA.TABLE_CONSTRAINTS in MySQL to audit primary keys, unique constraints, foreign keys, and check constraints.

---

## Overview

`INFORMATION_SCHEMA.TABLE_CONSTRAINTS` lists all named constraints in your MySQL database, including primary keys, unique keys, foreign keys, and check constraints. It provides a high-level view of data integrity enforcement without column details.

## Basic Query

```sql
SELECT
  CONSTRAINT_SCHEMA,
  TABLE_NAME,
  CONSTRAINT_NAME,
  CONSTRAINT_TYPE,
  ENFORCED
FROM information_schema.TABLE_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'myapp'
ORDER BY TABLE_NAME, CONSTRAINT_TYPE;
```

## Constraint Types

MySQL supports these constraint types:

```text
PRIMARY KEY   - uniquely identifies each row
UNIQUE        - enforces column value uniqueness
FOREIGN KEY   - enforces referential integrity
CHECK         - validates column value conditions (MySQL 8.0.16+)
```

## Finding Tables Without Primary Keys

```sql
SELECT t.TABLE_NAME
FROM information_schema.TABLES t
LEFT JOIN information_schema.TABLE_CONSTRAINTS tc
  ON t.TABLE_SCHEMA = tc.CONSTRAINT_SCHEMA
  AND t.TABLE_NAME = tc.TABLE_NAME
  AND tc.CONSTRAINT_TYPE = 'PRIMARY KEY'
WHERE t.TABLE_SCHEMA = 'myapp'
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND tc.TABLE_NAME IS NULL;
```

## Counting Constraints Per Table

```sql
SELECT
  TABLE_NAME,
  SUM(CONSTRAINT_TYPE = 'PRIMARY KEY') AS pk_count,
  SUM(CONSTRAINT_TYPE = 'UNIQUE') AS unique_count,
  SUM(CONSTRAINT_TYPE = 'FOREIGN KEY') AS fk_count,
  SUM(CONSTRAINT_TYPE = 'CHECK') AS check_count
FROM information_schema.TABLE_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'myapp'
GROUP BY TABLE_NAME
ORDER BY TABLE_NAME;
```

## Finding Tables with Foreign Keys

```sql
SELECT DISTINCT TABLE_NAME
FROM information_schema.TABLE_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'myapp'
  AND CONSTRAINT_TYPE = 'FOREIGN KEY'
ORDER BY TABLE_NAME;
```

## Finding Disabled Check Constraints

In MySQL 8.0+, check constraints can be NOT ENFORCED:

```sql
SELECT
  TABLE_NAME,
  CONSTRAINT_NAME,
  CONSTRAINT_TYPE
FROM information_schema.TABLE_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'myapp'
  AND CONSTRAINT_TYPE = 'CHECK'
  AND ENFORCED = 'NO';
```

## Joining with KEY_COLUMN_USAGE for Full Details

```sql
SELECT
  tc.TABLE_NAME,
  tc.CONSTRAINT_NAME,
  tc.CONSTRAINT_TYPE,
  GROUP_CONCAT(kcu.COLUMN_NAME ORDER BY kcu.ORDINAL_POSITION) AS columns
FROM information_schema.TABLE_CONSTRAINTS tc
JOIN information_schema.KEY_COLUMN_USAGE kcu
  ON tc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
  AND tc.TABLE_SCHEMA = kcu.TABLE_SCHEMA
  AND tc.TABLE_NAME = kcu.TABLE_NAME
WHERE tc.CONSTRAINT_SCHEMA = 'myapp'
GROUP BY tc.TABLE_NAME, tc.CONSTRAINT_NAME, tc.CONSTRAINT_TYPE
ORDER BY tc.TABLE_NAME, tc.CONSTRAINT_TYPE;
```

## Schema Health Check

```sql
-- Tables with no constraints at all
SELECT t.TABLE_NAME
FROM information_schema.TABLES t
LEFT JOIN information_schema.TABLE_CONSTRAINTS tc
  ON t.TABLE_SCHEMA = tc.CONSTRAINT_SCHEMA
  AND t.TABLE_NAME = tc.TABLE_NAME
WHERE t.TABLE_SCHEMA = 'myapp'
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND tc.TABLE_NAME IS NULL;
```

## Summary

`INFORMATION_SCHEMA.TABLE_CONSTRAINTS` provides a clear enumeration of all integrity constraints in your schema. By querying it, you can identify tables lacking primary keys, audit foreign key coverage, verify check constraint enforcement status, and generate schema health reports. Combine it with `KEY_COLUMN_USAGE` to get the complete picture including which columns each constraint covers.
