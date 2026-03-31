# How to Check the Character Set of a Table or Column in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Character Set, Collation, Information Schema, Inspection

Description: Learn multiple ways to inspect the character set and collation of MySQL tables and columns using SHOW commands and information_schema queries.

---

## Overview

Before migrating data or debugging collation errors, you need to know the current character set and collation settings for your tables and columns. MySQL provides several methods to retrieve this information.

## Check the Character Set of a Table

The quickest approach is `SHOW CREATE TABLE`:

```sql
SHOW CREATE TABLE users\G
```

Look for the `DEFAULT CHARSET` and `COLLATE` values at the end of the output:

```text
CREATE TABLE `users` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `email` varchar(255) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
```

## Query Table Collation via information_schema

For programmatic access or bulk inspection, query `information_schema.TABLES`:

```sql
SELECT
    TABLE_NAME,
    TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_NAME;
```

MySQL does not store the character set separately in this view; it is the prefix of the `TABLE_COLLATION` value (e.g., `utf8mb4` from `utf8mb4_unicode_ci`).

## Check Character Set and Collation for All Columns

Use `SHOW FULL COLUMNS` for a quick visual inspection:

```sql
SHOW FULL COLUMNS FROM users;
```

The `Collation` column shows the effective collation for each text column. Numeric and date columns display `NULL`.

For a more detailed or scriptable query:

```sql
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    CHARACTER_SET_NAME,
    COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME   = 'users'
ORDER BY ORDINAL_POSITION;
```

## Find Tables or Columns Using a Specific Character Set

Identify everything that still uses the legacy `utf8` (3-byte) character set:

```sql
-- Tables still using utf8
SELECT TABLE_NAME, TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_COLLATION LIKE 'utf8\_%';

-- Columns still using utf8
SELECT TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
  AND CHARACTER_SET_NAME = 'utf8';
```

## Check Database-Level Defaults

```sql
SELECT DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'your_database';
```

## Check Server and Session Defaults

```sql
SHOW VARIABLES LIKE 'character_set_%';
SHOW VARIABLES LIKE 'collation_%';
```

Key variables include `character_set_server`, `character_set_database`, and `character_set_client`.

## Summary

Use `SHOW CREATE TABLE` for a quick single-table check, and query `information_schema.TABLES` or `information_schema.COLUMNS` for systematic audits across your entire database. Identifying tables and columns that still use legacy `utf8` is the first step before any migration to `utf8mb4`.
