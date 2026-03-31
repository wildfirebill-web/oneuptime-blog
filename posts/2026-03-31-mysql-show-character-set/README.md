# How to Use SHOW CHARACTER SET in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Character Set, Encoding, Administration

Description: Learn how to use SHOW CHARACTER SET in MySQL to list available character sets, filter by pattern, and choose the right encoding for your database.

---

## What Is SHOW CHARACTER SET?

MySQL supports dozens of character sets - encoding schemes that define how text characters are stored and compared. The `SHOW CHARACTER SET` statement lists every character set available on your server, along with its default collation and the maximum number of bytes per character.

Choosing the right character set is critical for internationalization, storage efficiency, and correct text sorting.

## Basic Syntax

```sql
SHOW CHARACTER SET;
SHOW CHARACTER SET [LIKE 'pattern' | WHERE expr];
```

## Listing All Character Sets

```sql
SHOW CHARACTER SET;
```

Sample output:

```text
+----------+---------------------------------+---------------------+--------+
| Charset  | Description                     | Default collation   | Maxlen |
+----------+---------------------------------+---------------------+--------+
| armscii8 | ARMSCII-8 Armenian              | armscii8_general_ci |      1 |
| ascii    | US ASCII                        | ascii_general_ci    |      1 |
| big5     | Big5 Traditional Chinese        | big5_chinese_ci     |      2 |
| utf8mb4  | UTF-8 Unicode                   | utf8mb4_0900_ai_ci  |      4 |
| utf16    | UTF-16 Unicode                  | utf16_general_ci    |      4 |
+----------+---------------------------------+---------------------+--------+
```

## Filtering with LIKE

```sql
-- Find all UTF-related character sets
SHOW CHARACTER SET LIKE 'utf%';

-- Find latin character sets
SHOW CHARACTER SET LIKE 'latin%';
```

## Filtering with WHERE

```sql
-- Find character sets with a max length of 1 byte
SHOW CHARACTER SET WHERE Maxlen = 1;

-- Find character sets whose default collation is case-sensitive
SHOW CHARACTER SET WHERE `Default collation` LIKE '%_bin';
```

## Querying information_schema

For programmatic access or more filtering options, query the `information_schema`:

```sql
SELECT CHARACTER_SET_NAME, DEFAULT_COLLATE_NAME, MAXLEN
FROM information_schema.CHARACTER_SETS
WHERE CHARACTER_SET_NAME LIKE 'utf%'
ORDER BY CHARACTER_SET_NAME;
```

## Setting Character Sets at Different Levels

```sql
-- Set server default (my.cnf)
-- character-set-server = utf8mb4

-- Set database-level character set
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Alter existing database
ALTER DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Set column-level character set
CREATE TABLE users (
  name VARCHAR(100) CHARACTER SET utf8mb4
);
```

## Checking the Current Session Character Set

```sql
SHOW VARIABLES LIKE 'character_set_%';
```

```text
+--------------------------+------------------+
| Variable_name            | Value            |
+--------------------------+------------------+
| character_set_client     | utf8mb4          |
| character_set_connection | utf8mb4          |
| character_set_database   | utf8mb4          |
| character_set_results    | utf8mb4          |
| character_set_server     | utf8mb4          |
+--------------------------+------------------+
```

## Recommended Character Set

For modern applications, `utf8mb4` is the recommended character set. It supports the full Unicode range including emoji (unlike MySQL's older `utf8` which only covers the Basic Multilingual Plane):

```sql
-- Recommended defaults for new databases
CREATE DATABASE app_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

## Summary

`SHOW CHARACTER SET` gives you a complete view of every text encoding available on your MySQL server. Use it with `LIKE` or `WHERE` to narrow results, or query `information_schema.CHARACTER_SETS` for scripting. For almost all new projects, `utf8mb4` with `utf8mb4_unicode_ci` is the right choice, providing full Unicode coverage and consistent cross-platform collation.
