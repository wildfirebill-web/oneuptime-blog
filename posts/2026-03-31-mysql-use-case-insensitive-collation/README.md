# How to Use Case-Insensitive Collation in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Collation, Case Sensitivity, String Comparison, Query

Description: Learn how to configure and use case-insensitive collations in MySQL so that string comparisons and searches ignore letter case automatically.

---

## Overview

Case-insensitive (CI) collations allow MySQL to treat `Alice`, `alice`, and `ALICE` as identical strings during comparisons, sorting, and LIKE pattern matching. This is the default behavior for most MySQL installations and is the right choice for user-facing text such as usernames, email addresses, and product names.

## Identifying Case-Insensitive Collations

Collations ending in `_ci` are case-insensitive:

```sql
SHOW COLLATION WHERE Charset = 'utf8mb4' AND Collation LIKE '%_ci';
```

The most commonly used ones are:

| Collation | Notes |
|-----------|-------|
| `utf8mb4_general_ci` | Fast, older, less precise multilingual sorting |
| `utf8mb4_unicode_ci` | Unicode Collation Algorithm, good for most apps |
| `utf8mb4_0900_ai_ci` | Unicode 9.0, MySQL 8.0+ default, also accent-insensitive |

## Applying a Case-Insensitive Collation

At the database level:

```sql
CREATE DATABASE myapp
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

At the table level:

```sql
CREATE TABLE customers (
    id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    UNIQUE KEY uq_email (email)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

With this schema, inserting both `user@example.com` and `USER@EXAMPLE.COM` will trigger a duplicate key violation because the collation treats them as equal.

## Using COLLATE in a Query

To apply a case-insensitive comparison without changing the schema:

```sql
SELECT id, username
FROM users
WHERE username COLLATE utf8mb4_unicode_ci = 'alice';
```

This finds rows with `alice`, `Alice`, `ALICE`, and all other case variants.

## Case-Insensitive LIKE Searches

Under a `_ci` collation, `LIKE` is already case-insensitive:

```sql
SELECT * FROM products WHERE name LIKE '%laptop%';
-- Matches 'Laptop', 'LAPTOP', 'LapTop', etc.
```

If your column uses a `_bin` collation and you need a case-insensitive search without changing the schema, use `LOWER()`:

```sql
SELECT * FROM products WHERE LOWER(name) LIKE '%laptop%';
```

Note that `LOWER()` prevents index usage, so a functional index or schema change is preferable.

## Ensuring Consistent Behavior Across Connections

If the server default collation is different from your table collation, queries may behave unexpectedly. Confirm the effective collation at runtime:

```sql
SHOW VARIABLES LIKE 'collation_%';
```

Set it explicitly in the connection string or at session level if needed:

```sql
SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Summary

Case-insensitive collations like `utf8mb4_unicode_ci` or `utf8mb4_0900_ai_ci` are ideal for user-facing text fields. They simplify searching and deduplication without requiring application-level `lower()` calls. Apply them at the table or column level and keep them consistent across your schema to avoid collation mismatch errors.
