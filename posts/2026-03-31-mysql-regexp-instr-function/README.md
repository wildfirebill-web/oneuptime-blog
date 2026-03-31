# How to Use REGEXP_INSTR() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Regular Expression, Database

Description: Learn how MySQL's REGEXP_INSTR() function returns the position of a regular expression match within a string, enabling pattern-based position detection.

---

## What is REGEXP_INSTR()?

`REGEXP_INSTR()` is a MySQL 8.0+ function that returns the starting position of the first (or Nth) substring matching a regular expression pattern in a string. It returns 0 if no match is found. This function is useful when you need to locate where a pattern begins within a string - for example, finding the start of a date pattern, URL segment, or numeric sequence.

The syntax is:

```sql
REGEXP_INSTR(expr, pat [, pos [, occurrence [, return_option [, match_type]]]])
```

- `expr` - the string to search
- `pat` - the regular expression pattern
- `pos` - starting position for the search (default 1)
- `occurrence` - which match to return (default 1)
- `return_option` - 0 returns start position, 1 returns position after the match
- `match_type` - modifier string: `c` (case-sensitive), `i` (case-insensitive), `m` (multiline), etc.

## Basic Examples

```sql
SELECT REGEXP_INSTR('abc123def', '[0-9]+');
-- Result: 4  (digits start at position 4)

SELECT REGEXP_INSTR('hello world', 'world');
-- Result: 7

SELECT REGEXP_INSTR('no digits here', '[0-9]+');
-- Result: 0  (no match)
```

## Using pos to Start Searching Later

```sql
SELECT REGEXP_INSTR('abc123def456', '[0-9]+', 7);
-- Result: 10  (starts searching from position 7, finds second digit run)
```

## Finding the Nth Occurrence

```sql
SELECT REGEXP_INSTR('cat bat rat', 'at', 1, 2);
-- Result: 6  (position of the second 'at')

SELECT REGEXP_INSTR('cat bat rat', 'at', 1, 3);
-- Result: 10
```

## Using return_option = 1

When `return_option` is 1, the function returns the position just after the match ends:

```sql
SELECT REGEXP_INSTR('abc123def', '[0-9]+', 1, 1, 0);
-- Result: 4  (start of match)

SELECT REGEXP_INSTR('abc123def', '[0-9]+', 1, 1, 1);
-- Result: 7  (position after the match)
```

## Case-Insensitive Matching

```sql
SELECT REGEXP_INSTR('Hello World', 'hello', 1, 1, 0, 'i');
-- Result: 1

SELECT REGEXP_INSTR('Hello World', 'hello', 1, 1, 0, 'c');
-- Result: 0  (case-sensitive, no match)
```

## Practical Example - Locating Email Domain Start

```sql
SELECT
  email,
  REGEXP_INSTR(email, '@[a-zA-Z0-9.]+') AS domain_start_pos
FROM users;
```

## Finding First Digit in a Code

```sql
SELECT
  product_code,
  REGEXP_INSTR(product_code, '[0-9]') AS first_digit_position
FROM products;
```

## Using in WHERE Clause

```sql
SELECT *
FROM logs
WHERE REGEXP_INSTR(message, 'ERROR|FATAL') > 0;
```

This filters log rows where the message contains `ERROR` or `FATAL` anywhere.

## Practical Example - Detecting IPv4 Pattern Position

```sql
SELECT
  log_line,
  REGEXP_INSTR(log_line, '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}') AS ip_start
FROM access_logs
WHERE REGEXP_INSTR(log_line, '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}') > 0;
```

## Summary

`REGEXP_INSTR()` (MySQL 8.0+) returns the character position of a regex match, or 0 if no match exists. Optional parameters control the search start position, which occurrence to find, whether to return the start or end position, and case sensitivity. It is well suited for validating and locating patterns like IP addresses, dates, and numeric codes directly in SQL queries.
