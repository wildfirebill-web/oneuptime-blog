# How to Convert Between Character Sets in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Character Set, Conversion, Migration, Encoding

Description: Learn how to convert MySQL tables and columns from one character set to another, including the safe migration path from utf8 to utf8mb4.

---

## Overview

Changing a character set in MySQL does more than relabel data - it re-encodes the bytes stored on disk. The most common migration today is moving from the legacy `utf8` (3-byte) to `utf8mb4` (full 4-byte Unicode). Understanding the difference between "convert" and "change" operations prevents data loss.

## CONVERT TO vs. CHARACTER SET in ALTER TABLE

MySQL provides two related but distinct `ALTER TABLE` clauses:

```sql
-- Re-encodes existing data AND changes column definitions
ALTER TABLE articles CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Changes only the table DEFAULT for NEW columns; existing columns are unchanged
ALTER TABLE articles CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Always use `CONVERT TO` when you want to migrate all existing text data.

## Step-by-Step Migration: utf8 to utf8mb4

### 1. Back up the database first

```bash
mysqldump -u root -p --databases my_database > backup_before_migration.sql
```

### 2. Check current state

```sql
SELECT TABLE_NAME, TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'my_database'
  AND TABLE_COLLATION LIKE 'utf8\_%';
```

### 3. Alter the database default

```sql
ALTER DATABASE my_database
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

### 4. Convert each table

```sql
ALTER TABLE articles
    CONVERT TO CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

ALTER TABLE users
    CONVERT TO CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

### 5. Generate statements for all tables at once

```sql
SELECT CONCAT(
    'ALTER TABLE `', TABLE_NAME,
    '` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
) AS statement
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'my_database'
  AND TABLE_TYPE   = 'BASE TABLE';
```

## Converting a Single Column

```sql
ALTER TABLE products
    MODIFY COLUMN description TEXT
        CHARACTER SET utf8mb4
        COLLATE utf8mb4_unicode_ci;
```

## Using the CONVERT() Function for In-Query Conversion

The `CONVERT()` function can coerce string expressions to a different character set at query time without altering the schema:

```sql
SELECT CONVERT(legacy_field USING utf8mb4) AS modern_text
FROM legacy_table;
```

This is useful for one-off queries or data validation before committing to a schema change.

## Verifying After Migration

```sql
SELECT TABLE_NAME, TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'my_database';

SELECT COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'my_database'
  AND TABLE_NAME   = 'articles';
```

## Summary

Use `ALTER TABLE ... CONVERT TO CHARACTER SET` to re-encode all data in a table and update column definitions. Always back up before running bulk conversions, and verify column-level settings in `information_schema.COLUMNS` after migration. The `CONVERT()` function provides a non-destructive way to preview or temporarily re-encode data in queries.
