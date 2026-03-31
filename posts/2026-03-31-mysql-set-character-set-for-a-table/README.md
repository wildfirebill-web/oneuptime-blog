# How to Set the Character Set for a MySQL Table

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Character Set, Table, Encoding, Schema

Description: Learn how to set or change the character set for a MySQL table using CREATE TABLE and ALTER TABLE syntax with practical examples.

---

## Overview

MySQL stores text using a character set that defines which characters can be stored. Setting the correct character set at the table level ensures that all string columns inherit a consistent encoding by default. The most common choice today is `utf8mb4`, which supports the full Unicode range including emoji and supplementary characters.

## Setting Character Set at Table Creation

When creating a new table, specify the character set using the `CHARACTER SET` (or `CHARSET`) option:

```sql
CREATE TABLE articles (
    id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title    VARCHAR(255),
    body     TEXT,
    created  DATETIME
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

All `VARCHAR`, `TEXT`, and `CHAR` columns in this table will inherit `utf8mb4` unless a column-level override is given.

## Changing the Character Set of an Existing Table

Use `ALTER TABLE ... CONVERT TO CHARACTER SET` to re-encode all text columns in one step:

```sql
ALTER TABLE articles
    CONVERT TO CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

This command rewrites every string column, so run it during a maintenance window or use an online DDL approach (such as `pt-online-schema-change`) on large tables.

If you only want to change the table default without re-encoding existing columns, use:

```sql
ALTER TABLE articles
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

New columns added after this command will use the new default, but existing columns keep their original character set.

## Verifying the Table Character Set

After creation or alteration, confirm the settings with:

```sql
SHOW CREATE TABLE articles\G
```

Look for `DEFAULT CHARSET=utf8mb4` in the output. You can also query `information_schema`:

```sql
SELECT TABLE_NAME, TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME   = 'articles';
```

## Common Mistakes to Avoid

- **Setting `CHARACTER SET utf8`** - MySQL's `utf8` alias only covers the Basic Multilingual Plane (3 bytes per character). Use `utf8mb4` instead.
- **Mismatched collations** - If two columns being compared use different collations MySQL raises an `Illegal mix of collations` error. Always use a consistent collation across related tables.
- **Forgetting `COLLATE`** - Changing the character set without specifying a collation leaves MySQL choosing the default, which may differ across server versions.

## Practical Script

Here is a snippet that alters every table in a database to `utf8mb4`:

```sql
SELECT CONCAT(
    'ALTER TABLE `', TABLE_NAME, '` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
) AS alter_statement
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_TYPE   = 'BASE TABLE';
```

Copy the output, review it, and run each statement individually.

## Summary

Setting the character set at the table level controls the default encoding for all string columns in that table. Use `CREATE TABLE ... CHARACTER SET utf8mb4` for new tables, and `ALTER TABLE ... CONVERT TO CHARACTER SET utf8mb4` to migrate existing tables. Always pair the character set with an explicit collation to avoid unexpected sort and comparison behavior.
