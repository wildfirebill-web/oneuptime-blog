# How to Use REGEXP_REPLACE() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Regexp Replace, Regular Expressions, String Functions, Sql

Description: Learn how to use the REGEXP_REPLACE() function in MySQL 8.0+ to replace parts of a string that match a regular expression pattern with a replacement string.

---

## Introduction

`REGEXP_REPLACE()` is a MySQL 8.0+ function that replaces occurrences of a substring that matches a regular expression pattern with a specified replacement string. It is the regex-powered counterpart to `REPLACE()` and enables complex string substitutions that simple text replacement cannot achieve.

## Basic Syntax

```sql
REGEXP_REPLACE(string, pattern, replacement [, position [, occurrence [, match_type]]])
```

- `string`: the input string to search.
- `pattern`: the regular expression pattern to match.
- `replacement`: the replacement string.
- `position`: starting position (default 1).
- `occurrence`: which occurrence to replace (0 = all, 1 = first, 2 = second, etc.; default 0).
- `match_type`: match modifiers (e.g., 'i' for case-insensitive, 'm' for multiline).

## Basic Examples

```sql
-- Remove all digits from a string
SELECT REGEXP_REPLACE('abc123def456', '[0-9]', '');
-- Returns: 'abcdef'

-- Replace multiple spaces with a single space
SELECT REGEXP_REPLACE('hello   world  again', '  +', ' ');
-- Returns: 'hello world again'

-- Remove HTML tags
SELECT REGEXP_REPLACE('<p>Hello <b>World</b></p>', '<[^>]+>', '');
-- Returns: 'Hello World'
```

## Replace All vs Specific Occurrence

```sql
-- Replace all occurrences (default)
SELECT REGEXP_REPLACE('cat bat hat mat', '[cbhm]at', 'XXX');
-- Returns: 'XXX XXX XXX XXX'

-- Replace only the first occurrence
SELECT REGEXP_REPLACE('cat bat hat mat', '[cbhm]at', 'XXX', 1, 1);
-- Returns: 'XXX bat hat mat'

-- Replace only the second occurrence
SELECT REGEXP_REPLACE('cat bat hat mat', '[cbhm]at', 'XXX', 1, 2);
-- Returns: 'cat XXX hat mat'
```

## Case-Insensitive Replacement

```sql
-- By default, REGEXP_REPLACE is case-sensitive
SELECT REGEXP_REPLACE('Hello World HELLO', 'hello', 'Hi');
-- Returns: 'Hello World HELLO' (no change - case sensitive)

-- Use 'i' match type for case-insensitive
SELECT REGEXP_REPLACE('Hello World HELLO', 'hello', 'Hi', 1, 0, 'i');
-- Returns: 'Hi World Hi'
```

## Normalizing Phone Numbers

```sql
-- Remove all non-numeric characters from phone numbers
SELECT REGEXP_REPLACE(phone, '[^0-9]', '') AS clean_phone
FROM contacts;

-- Input: '(555) 123-4567' or '+1-555-123-4567'
-- Output: '5551234567' or '15551234567'
```

## Cleaning Up Extra Whitespace

```sql
-- Trim and normalize internal whitespace
SELECT TRIM(REGEXP_REPLACE(description, '\\s+', ' ')) AS clean_description
FROM products;
```

## Masking Sensitive Data

```sql
-- Mask all but the last 4 digits of a credit card
SELECT REGEXP_REPLACE('4111-1111-1111-1234', '\\d', 'X', 1, 0) AS masked;
-- Returns: 'XXXX-XXXX-XXXX-XXXX'

-- Keep last 4 visible (more complex pattern needed)
SELECT CONCAT('****-****-****-', RIGHT(REGEXP_REPLACE(card_number, '[^0-9]', ''), 4))
AS masked_card
FROM payment_methods;
```

## Remove Special Characters

```sql
-- Keep only alphanumeric characters
SELECT REGEXP_REPLACE(username, '[^a-zA-Z0-9]', '') AS clean_username
FROM users;
```

## Updating Data with REGEXP_REPLACE

Use it in an UPDATE statement to clean existing data:

```sql
-- Remove leading zeros from ZIP codes stored as strings
UPDATE addresses
SET zip_code = REGEXP_REPLACE(zip_code, '^0+', '')
WHERE zip_code REGEXP '^0';

-- Normalize phone numbers in bulk
UPDATE contacts
SET phone = REGEXP_REPLACE(phone, '[^0-9]', '');
```

## REGEXP_REPLACE vs REPLACE

```sql
-- REPLACE: exact string match only
SELECT REPLACE('cat bat hat', 'at', 'oo');
-- Returns: 'coo boo hoo'

-- REGEXP_REPLACE: pattern-based
SELECT REGEXP_REPLACE('cat bat hat', '[cbh]at', 'dog');
-- Returns: 'dog dog dog'
```

Use `REPLACE` for simple fixed-string substitutions. Use `REGEXP_REPLACE` when you need pattern-based matching.

## Common Regular Expression Patterns

```sql
-- Remove all whitespace
REGEXP_REPLACE(col, '\\s+', '')

-- Replace repeated punctuation
REGEXP_REPLACE(col, '[!?.]{2,}', '.')

-- Remove URLs
REGEXP_REPLACE(col, 'https?://[^\\s]+', '[link]')

-- Normalize date separators
REGEXP_REPLACE(col, '[/\\-.]', '-')
```

## Summary

`REGEXP_REPLACE()` is a powerful MySQL 8.0+ function for pattern-based string replacement using regular expressions. It supports replacing all or specific occurrences, case-insensitive matching, and complex patterns that are impossible with simple `REPLACE()`. Common uses include cleaning phone numbers, removing HTML tags, normalizing whitespace, and sanitizing user input.
