# How to Query INFORMATION_SCHEMA.CHARACTER_SETS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Character Set, Encoding, Database Administration

Description: Learn how to query INFORMATION_SCHEMA.CHARACTER_SETS in MySQL to explore available character sets, their default collations, and maximum byte lengths.

---

## What Is INFORMATION_SCHEMA.CHARACTER_SETS?

The `INFORMATION_SCHEMA.CHARACTER_SETS` table lists all character sets supported by your MySQL server. A character set defines how characters are stored and encoded at the byte level. MySQL supports dozens of character sets ranging from single-byte options like `latin1` to multi-byte sets like `utf8mb4`, which supports the full Unicode range including emoji.

Querying this table helps you understand which character sets are available, choose the right encoding for your use case, and verify that your server supports the required character set before creating databases or tables.

## Columns in CHARACTER_SETS

- `CHARACTER_SET_NAME` - the character set identifier (e.g., `utf8mb4`)
- `DEFAULT_COLLATE_NAME` - the default collation for the character set
- `DESCRIPTION` - a human-readable description
- `MAXLEN` - the maximum number of bytes per character

## List All Available Character Sets

```sql
SELECT
    CHARACTER_SET_NAME,
    DEFAULT_COLLATE_NAME,
    MAXLEN
FROM INFORMATION_SCHEMA.CHARACTER_SETS
ORDER BY CHARACTER_SET_NAME;
```

## Find Multi-Byte Character Sets

```sql
SELECT
    CHARACTER_SET_NAME,
    DEFAULT_COLLATE_NAME,
    MAXLEN,
    DESCRIPTION
FROM INFORMATION_SCHEMA.CHARACTER_SETS
WHERE MAXLEN > 1
ORDER BY MAXLEN DESC, CHARACTER_SET_NAME;
```

## Find Unicode-Compatible Character Sets

```sql
SELECT
    CHARACTER_SET_NAME,
    DEFAULT_COLLATE_NAME,
    MAXLEN
FROM INFORMATION_SCHEMA.CHARACTER_SETS
WHERE CHARACTER_SET_NAME LIKE 'utf%'
   OR CHARACTER_SET_NAME LIKE 'ucs%'
ORDER BY CHARACTER_SET_NAME;
```

## Check if a Specific Character Set Is Available

Before creating a database with a specific encoding, verify it exists:

```sql
SELECT CHARACTER_SET_NAME, MAXLEN
FROM INFORMATION_SCHEMA.CHARACTER_SETS
WHERE CHARACTER_SET_NAME = 'utf8mb4';
```

Expected output:

```text
+--------------------+--------+
| CHARACTER_SET_NAME | MAXLEN |
+--------------------+--------+
| utf8mb4            |      4 |
+--------------------+--------+
```

## Find All Single-Byte Character Sets

Single-byte character sets are memory-efficient but limited in range:

```sql
SELECT
    CHARACTER_SET_NAME,
    DESCRIPTION
FROM INFORMATION_SCHEMA.CHARACTER_SETS
WHERE MAXLEN = 1
ORDER BY CHARACTER_SET_NAME;
```

## Recommended Character Set: utf8mb4

For new MySQL databases, `utf8mb4` is the recommended choice. It stores up to 4 bytes per character and supports the full Unicode standard, including supplementary characters and emoji:

```sql
CREATE DATABASE myapp
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Verify the setting took effect:

```sql
SELECT
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME
FROM INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME = 'myapp';
```

## Why Not Use utf8 (3-Byte)?

MySQL's `utf8` character set is actually `utf8mb3` - a 3-byte subset of Unicode that cannot store 4-byte characters like many emoji or rare CJK ideographs. Always prefer `utf8mb4` over `utf8` in new deployments.

```sql
-- Confirm utf8mb4 is 4 bytes, utf8 is 3 bytes
SELECT CHARACTER_SET_NAME, MAXLEN
FROM INFORMATION_SCHEMA.CHARACTER_SETS
WHERE CHARACTER_SET_NAME IN ('utf8', 'utf8mb4');
```

## Summary

`INFORMATION_SCHEMA.CHARACTER_SETS` is your reference for understanding which character encodings MySQL supports and their storage characteristics. Use it to validate encoding options before designing schemas, and always favor `utf8mb4` for modern applications requiring full Unicode support.
