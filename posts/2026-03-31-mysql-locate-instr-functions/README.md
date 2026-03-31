# How to Use LOCATE() and INSTR() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, String Function, Database

Description: Learn how LOCATE() and INSTR() find the position of a substring within a string in MySQL, including starting offset search, NULL handling, and practical extraction patterns.

---

`LOCATE()` and `INSTR()` both return the position of the first occurrence of a substring within a string. `LOCATE` is more flexible because it accepts an optional starting position; `INSTR` is a two-argument shorthand.

## Syntax

```sql
LOCATE(substr, str)
LOCATE(substr, str, pos)   -- search starting from position pos

INSTR(str, substr)         -- same as LOCATE(substr, str) but arguments are reversed
```

All return the character position (1-indexed). Return `0` if `substr` is not found. Return `NULL` if any argument is `NULL`.

## Basic usage

```sql
SELECT LOCATE('World', 'Hello World');       -- 7
SELECT LOCATE('o',     'Hello World');       -- 5  (first 'o')
SELECT LOCATE('xyz',   'Hello World');       -- 0  (not found)

SELECT INSTR('Hello World', 'World');        -- 7
SELECT INSTR('Hello World', 'o');            -- 5
```

## Finding the second occurrence using the pos argument

```sql
-- Find position of second 'o' in 'Hello World'
SELECT LOCATE('o', 'Hello World', 6);  -- 8  (search starts at position 6)

-- Find position of third 'a' in 'banana'
SELECT LOCATE('a', 'banana',
    LOCATE('a', 'banana',
        LOCATE('a', 'banana') + 1
    ) + 1
);  -- 6
```

## Checking whether a substring exists

```sql
-- LOCATE returns 0 if not found, use in WHERE
SELECT email FROM users WHERE LOCATE('@', email) > 0;

-- INSTR equivalent
SELECT email FROM users WHERE INSTR(email, '@') > 0;

-- Simpler with LIKE
SELECT email FROM users WHERE email LIKE '%@%';
```

## Extracting a substring using LOCATE

Combine `LOCATE` with `SUBSTRING` to extract dynamic substrings:

```sql
-- Extract the domain from an email address
SELECT
    email,
    SUBSTRING(email, LOCATE('@', email) + 1) AS domain
FROM users;
-- 'alice@example.com' -> 'example.com'

-- Extract the username part
SELECT
    email,
    LEFT(email, LOCATE('@', email) - 1) AS username
FROM users;
-- 'alice@example.com' -> 'alice'
```

## Extracting between two delimiters

```sql
-- Extract the value between the first and second '/' in a URL path
-- e.g. '/api/users/42' -> 'users'
SELECT
    path,
    SUBSTRING(
        path,
        LOCATE('/', path, 2) + 1,
        LOCATE('/', path, LOCATE('/', path, 2) + 1) - LOCATE('/', path, 2) - 1
    ) AS segment
FROM api_logs;
```

## Validating data format

```sql
-- Check that phone numbers contain a '-' separator
SELECT phone FROM contacts WHERE LOCATE('-', phone) = 0;

-- Find emails missing '@'
SELECT email FROM users WHERE LOCATE('@', email) = 0;
```

## Case sensitivity

`LOCATE` and `INSTR` are case-insensitive when using a case-insensitive collation (the default `utf8mb4_0900_ai_ci`):

```sql
SELECT LOCATE('hello', 'Hello World');  -- 1 (case-insensitive by default)
SELECT LOCATE('WORLD', 'Hello World');  -- 7 (case-insensitive by default)
```

For a case-sensitive search, use a binary collation:

```sql
SELECT LOCATE('hello', 'Hello World' COLLATE utf8mb4_bin);  -- 0 (not found)
SELECT LOCATE('Hello', 'Hello World' COLLATE utf8mb4_bin);  -- 1
```

## LOCATE vs INSTR argument order

```sql
-- LOCATE: search string first, then source string
SELECT LOCATE('ell', 'Hello');   -- 2

-- INSTR: source string first, then search string (reversed)
SELECT INSTR('Hello', 'ell');    -- 2
```

Both return the same result. The argument order difference mirrors `LIKE` (source first) vs SQL's `POSITION` function. `LOCATE` with the optional third argument is the more powerful of the two.

## Using POSITION as an alternative

```sql
-- POSITION is standard SQL, equivalent to LOCATE without pos argument
SELECT POSITION('World' IN 'Hello World');  -- 7
```

## Practical example: parse a log line

```sql
CREATE TABLE logs (
    log_id  INT PRIMARY KEY,
    line    TEXT
    -- example: '2026-03-31 ERROR Connection refused'
);

SELECT
    log_id,
    LEFT(line, 10)                                  AS log_date,
    SUBSTRING(line, 12, LOCATE(' ', line, 12) - 12) AS level,
    SUBSTRING(line, LOCATE(' ', line, 12) + 1)      AS message
FROM logs;
```

## Summary

`LOCATE(substr, str [, pos])` and `INSTR(str, substr)` both return the 1-indexed character position of the first occurrence of a substring. `LOCATE` is preferred when you need to search starting from a specific offset to find the second or later occurrence. Combine these functions with `SUBSTRING`, `LEFT`, and `RIGHT` to extract dynamic fields from formatted strings. Both functions are case-insensitive under the default MySQL collation.
