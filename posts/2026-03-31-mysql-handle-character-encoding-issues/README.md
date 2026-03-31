# How to Handle Character Encoding Issues in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Character Encoding, UTF-8, Collation, Database

Description: Learn how to diagnose and fix character encoding issues in MySQL, including mojibake, incorrect string value errors, and collation mismatches.

---

## Understanding Character Sets and Collations

MySQL uses character sets to define which characters can be stored, and collations to define how characters are compared and sorted. The most common source of encoding issues is a mismatch between the client, connection, database, table, or column character sets.

Check the current settings:

```sql
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';
```

## Setting UTF-8 at Every Level

MySQL has two UTF-8 encodings: `utf8` (which is actually a 3-byte subset) and `utf8mb4` (true 4-byte UTF-8 that supports emojis and supplementary characters). Always use `utf8mb4`.

```sql
-- Set server defaults in my.cnf
-- [mysqld]
-- character-set-server = utf8mb4
-- collation-server = utf8mb4_unicode_ci

-- At the database level
CREATE DATABASE mydb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- At the table level
CREATE TABLE messages (
  id INT AUTO_INCREMENT PRIMARY KEY,
  body TEXT NOT NULL
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Setting the Connection Character Set

The connection character set must match the encoding of data your application sends:

```sql
SET NAMES utf8mb4;
-- Equivalent to:
-- SET character_set_client = utf8mb4;
-- SET character_set_connection = utf8mb4;
-- SET character_set_results = utf8mb4;
```

In application connection strings, add the charset parameter:

```bash
mysql://user:pass@host/dbname?charset=utf8mb4
```

## Fixing Mojibake - Garbled Text

Mojibake occurs when text encoded in one charset is stored or read as another. A common case is UTF-8 data stored as Latin-1.

Diagnose by checking the hex values:

```sql
SELECT HEX(body) FROM messages WHERE id = 1;
-- If you see C3A9 for 'e' with accent, data is UTF-8 but column may be Latin-1
```

To fix an existing column:

```sql
ALTER TABLE messages
  MODIFY COLUMN body TEXT
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

If the data itself is stored with the wrong encoding, use a two-step conversion:

```sql
ALTER TABLE messages MODIFY COLUMN body BLOB;
ALTER TABLE messages MODIFY COLUMN body TEXT
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Fixing "Incorrect string value" Errors

This error appears when trying to insert a character (like an emoji) into a column that does not support 4-byte UTF-8:

```sql
-- Error: Incorrect string value: '\xF0\x9F\x98\x8A' for column 'body'
-- Fix: upgrade the column to utf8mb4
ALTER TABLE messages
  MODIFY COLUMN body TEXT
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Handling Collation Mismatches in Joins

Joining columns with different collations causes an "Illegal mix of collations" error:

```sql
-- Fix by casting to a common collation
SELECT a.name, b.name
FROM table_a a
JOIN table_b b
  ON a.name COLLATE utf8mb4_unicode_ci = b.name COLLATE utf8mb4_unicode_ci;
```

## Summary

Character encoding issues in MySQL typically stem from mismatches between the connection, column, and data encoding. Always use `utf8mb4` throughout your stack and set `SET NAMES utf8mb4` at connection time. For existing data with mojibake, use a BLOB intermediary during conversion to avoid double-encoding.
