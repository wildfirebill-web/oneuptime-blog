# How to Use REGEXP_LIKE() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Regexp Like, Regular Expressions, String Functions, Sql

Description: Learn how to use REGEXP_LIKE() in MySQL 8.0+ to test whether a string matches a regular expression pattern, with examples for validation and filtering.

---

## Introduction

`REGEXP_LIKE()` is a MySQL 8.0+ function that tests whether a string matches a regular expression pattern. It returns 1 (TRUE) if the string matches, and 0 (FALSE) otherwise. It is the functional equivalent of the `REGEXP` and `RLIKE` operators but offers additional control through match type flags.

## Basic Syntax

```sql
REGEXP_LIKE(string, pattern [, match_type])
```

- `string`: the input string to test.
- `pattern`: the regular expression pattern.
- `match_type`: optional modifiers ('i' = case-insensitive, 'c' = case-sensitive, 'm' = multiline, 'n' = dot matches newline, 'u' = Unix line endings).

## Basic Examples

```sql
SELECT REGEXP_LIKE('Hello World', 'world');
-- Returns: 0 (case-sensitive by default)

SELECT REGEXP_LIKE('Hello World', 'world', 'i');
-- Returns: 1 (case-insensitive match)

SELECT REGEXP_LIKE('abc123', '^[a-z]+[0-9]+$');
-- Returns: 1 (matches pattern: letters followed by digits)

SELECT REGEXP_LIKE('abc123', '^[0-9]+$');
-- Returns: 0 (does not match: not all digits)
```

## REGEXP_LIKE vs REGEXP Operator

```sql
-- REGEXP operator (older syntax)
SELECT name FROM users WHERE name REGEXP '^[A-Z]';

-- REGEXP_LIKE function (8.0+, more control)
SELECT name FROM users WHERE REGEXP_LIKE(name, '^[A-Z]');

-- With case-insensitive flag (only possible with REGEXP_LIKE)
SELECT name FROM users WHERE REGEXP_LIKE(name, '^[a-z]', 'i');
```

## Email Validation

```sql
SELECT email,
  REGEXP_LIKE(email, '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$') AS is_valid
FROM users;
```

Filter only valid emails:

```sql
SELECT id, email
FROM users
WHERE REGEXP_LIKE(email, '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$');
```

## Phone Number Validation

```sql
-- Match 10-digit US phone numbers (digits only)
SELECT REGEXP_LIKE('5551234567', '^[0-9]{10}$');
-- Returns: 1

-- Match formatted phone number
SELECT REGEXP_LIKE('(555) 123-4567', '^\\([0-9]{3}\\) [0-9]{3}-[0-9]{4}$');
-- Returns: 1
```

Find rows with invalid phone formats:

```sql
SELECT id, phone
FROM contacts
WHERE NOT REGEXP_LIKE(phone, '^[0-9]{10}$')
  AND phone IS NOT NULL;
```

## Filtering by Pattern in Queries

Find product codes matching a specific pattern (2 letters, dash, 4 digits):

```sql
SELECT id, code, name
FROM products
WHERE REGEXP_LIKE(code, '^[A-Z]{2}-[0-9]{4}$');
```

Find users with names containing only letters and spaces:

```sql
SELECT id, name
FROM users
WHERE REGEXP_LIKE(name, '^[a-zA-Z ]+$');
```

## Case-Insensitive Matching

```sql
-- Case-sensitive (default)
SELECT REGEXP_LIKE('MySQL', 'mysql');
-- Returns: 0

-- Case-insensitive
SELECT REGEXP_LIKE('MySQL', 'mysql', 'i');
-- Returns: 1
```

## Multiline Mode

In multiline mode, `^` and `$` match the start and end of each line, not just the entire string.

```sql
SELECT REGEXP_LIKE('line1\nline2\nline3', '^line2$', 'm');
-- Returns: 1
```

## Finding Rows with Numeric-Only Values

```sql
SELECT id, postal_code
FROM addresses
WHERE REGEXP_LIKE(postal_code, '^[0-9]+$');
```

## Finding Rows with Mixed Content (Data Quality Check)

```sql
-- Find product descriptions that contain HTML tags
SELECT id, description
FROM products
WHERE REGEXP_LIKE(description, '<[a-z][^>]*>', 'i');
```

## REGEXP_LIKE in CHECK Constraints (MySQL 8.0.16+)

```sql
CREATE TABLE contacts (
  id INT PRIMARY KEY,
  email VARCHAR(255),
  CONSTRAINT chk_email CHECK (REGEXP_LIKE(email, '^[^@]+@[^@]+\\.[^@]+$'))
);
```

## Common Patterns for REGEXP_LIKE

```sql
-- Only digits
REGEXP_LIKE(col, '^[0-9]+$')

-- Only letters
REGEXP_LIKE(col, '^[a-zA-Z]+$')

-- Alphanumeric
REGEXP_LIKE(col, '^[a-zA-Z0-9]+$')

-- Starts with uppercase
REGEXP_LIKE(col, '^[A-Z]')

-- Valid URL (simplified)
REGEXP_LIKE(col, '^https?://', 'i')

-- Contains a word
REGEXP_LIKE(col, '\\bword\\b', 'i')
```

## Summary

`REGEXP_LIKE()` tests whether a string matches a regular expression pattern in MySQL 8.0+. It provides more control than the `REGEXP` operator through optional match flags for case sensitivity and multiline matching. Common uses include email validation, pattern-based filtering, data quality checks, and enforcing format constraints.
