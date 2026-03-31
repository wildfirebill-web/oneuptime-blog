# How to Query INFORMATION_SCHEMA.COLLATIONS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Collation, Character Set, Database Administration

Description: Learn how to query INFORMATION_SCHEMA.COLLATIONS in MySQL to explore available collations, their character sets, and sorting properties.

---

## What Is INFORMATION_SCHEMA.COLLATIONS?

The `INFORMATION_SCHEMA.COLLATIONS` table lists all collations supported by your MySQL server. A collation defines the rules used to compare and sort character strings. Every character set has at least one collation, and most have many - covering case sensitivity, accent sensitivity, and language-specific ordering rules.

Choosing the right collation is critical: it affects how `ORDER BY`, `WHERE`, `GROUP BY`, and `JOIN` operations behave when comparing strings.

## Columns in COLLATIONS

- `COLLATION_NAME` - the collation identifier (e.g., `utf8mb4_unicode_ci`)
- `CHARACTER_SET_NAME` - the character set this collation belongs to
- `ID` - numeric identifier
- `IS_DEFAULT` - `Yes` if this is the default collation for the character set
- `IS_COMPILED` - `Yes` if compiled into the server
- `SORTLEN` - memory used for sorting
- `PAD_ATTRIBUTE` - `PAD SPACE` or `NO PAD` (MySQL 8.0+)

## List All Collations

```sql
SELECT
    COLLATION_NAME,
    CHARACTER_SET_NAME,
    IS_DEFAULT
FROM INFORMATION_SCHEMA.COLLATIONS
ORDER BY CHARACTER_SET_NAME, COLLATION_NAME;
```

## Find All Collations for utf8mb4

```sql
SELECT
    COLLATION_NAME,
    IS_DEFAULT,
    PAD_ATTRIBUTE
FROM INFORMATION_SCHEMA.COLLATIONS
WHERE CHARACTER_SET_NAME = 'utf8mb4'
ORDER BY COLLATION_NAME;
```

## Find Default Collations for Each Character Set

```sql
SELECT
    CHARACTER_SET_NAME,
    COLLATION_NAME
FROM INFORMATION_SCHEMA.COLLATIONS
WHERE IS_DEFAULT = 'Yes'
ORDER BY CHARACTER_SET_NAME;
```

## Understand Case-Insensitive vs Case-Sensitive Collations

Collation names follow a naming convention:
- `_ci` suffix = case-insensitive
- `_cs` suffix = case-sensitive
- `_bin` suffix = binary comparison

```sql
SELECT COLLATION_NAME
FROM INFORMATION_SCHEMA.COLLATIONS
WHERE CHARACTER_SET_NAME = 'utf8mb4'
  AND (
    COLLATION_NAME LIKE '%_ci'
    OR COLLATION_NAME LIKE '%_cs'
    OR COLLATION_NAME LIKE '%_bin'
  )
ORDER BY COLLATION_NAME;
```

## Check Whether a Collation Uses PAD SPACE or NO PAD

In MySQL 8.0+, trailing space handling differs between collations:

```sql
SELECT
    COLLATION_NAME,
    PAD_ATTRIBUTE
FROM INFORMATION_SCHEMA.COLLATIONS
WHERE CHARACTER_SET_NAME = 'utf8mb4'
ORDER BY PAD_ATTRIBUTE, COLLATION_NAME;
```

`NO PAD` collations treat `'hello'` and `'hello '` as different strings. `PAD SPACE` collations treat them as equal.

## Choosing a Collation for New Tables

For most English-language applications, `utf8mb4_unicode_ci` provides good multilingual support. For strict binary ordering or case-sensitive matching, use `utf8mb4_bin`:

```sql
CREATE TABLE articles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci,
    content TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
);
```

## Verify a Column's Current Collation

```sql
SELECT
    COLUMN_NAME,
    CHARACTER_SET_NAME,
    COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'articles';
```

## Summary

`INFORMATION_SCHEMA.COLLATIONS` gives you a complete inventory of sorting and comparison rules available in MySQL. Use it to pick the right collation before schema design, understand case and accent sensitivity implications, and avoid subtle bugs caused by mismatched collations in joins or comparisons between string columns.
