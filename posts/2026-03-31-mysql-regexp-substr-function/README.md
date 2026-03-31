# How to Use REGEXP_SUBSTR() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Regular Expression, Database

Description: Learn how MySQL's REGEXP_SUBSTR() function extracts the substring matching a regular expression, with options for occurrence and case sensitivity.

---

## What is REGEXP_SUBSTR()?

`REGEXP_SUBSTR()` is a MySQL 8.0+ function that returns the portion of a string that matches a regular expression pattern. If no match is found, it returns `NULL`. This makes it ideal for extracting structured data embedded in free-text columns - such as phone numbers, IP addresses, dates, or product codes.

The syntax is:

```sql
REGEXP_SUBSTR(expr, pat [, pos [, occurrence [, match_type]]])
```

- `expr` - the string to search
- `pat` - the regular expression
- `pos` - starting position (default 1)
- `occurrence` - which match to extract (default 1)
- `match_type` - modifier string: `i` (case-insensitive), `c` (case-sensitive), `m` (multiline)

## Basic Examples

```sql
SELECT REGEXP_SUBSTR('Order #12345 placed', '[0-9]+');
-- Result: 12345

SELECT REGEXP_SUBSTR('hello world', 'w[a-z]+');
-- Result: world

SELECT REGEXP_SUBSTR('no numbers', '[0-9]+');
-- Result: NULL
```

## Extracting Multiple Occurrences

Use the `occurrence` parameter to get the 2nd, 3rd, etc. match:

```sql
SELECT REGEXP_SUBSTR('abc 123 def 456', '[0-9]+', 1, 1);
-- Result: 123

SELECT REGEXP_SUBSTR('abc 123 def 456', '[0-9]+', 1, 2);
-- Result: 456
```

## Case-Insensitive Matching

```sql
SELECT REGEXP_SUBSTR('Status: Active', 'active', 1, 1, 'i');
-- Result: Active
```

## Practical Example - Extracting IP Addresses from Logs

```sql
SELECT
  log_line,
  REGEXP_SUBSTR(log_line, '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}') AS ip_address
FROM access_logs
WHERE REGEXP_SUBSTR(log_line, '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}') IS NOT NULL;
```

## Extracting Email Addresses

```sql
SELECT
  raw_text,
  REGEXP_SUBSTR(raw_text, '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}') AS email
FROM contact_submissions;
```

## Extracting Dates in YYYY-MM-DD Format

```sql
SELECT
  note,
  REGEXP_SUBSTR(note, '[0-9]{4}-[0-9]{2}-[0-9]{2}') AS extracted_date
FROM meeting_notes;
```

## Using in UPDATE

You can use `REGEXP_SUBSTR()` to populate a new column from an existing free-text column:

```sql
UPDATE orders
SET tracking_number = REGEXP_SUBSTR(raw_notes, '[A-Z]{2}[0-9]{9}[A-Z]{2}')
WHERE tracking_number IS NULL;
```

## Combining with REGEXP_INSTR()

When you need both the position and the matched value, combine both functions:

```sql
SELECT
  content,
  REGEXP_INSTR(content, '[0-9]+')  AS digit_pos,
  REGEXP_SUBSTR(content, '[0-9]+') AS digit_match
FROM documents;
```

## Extracting Version Numbers

```sql
SELECT
  software_name,
  REGEXP_SUBSTR(version_string, '[0-9]+\\.[0-9]+\\.[0-9]+') AS semver
FROM installed_packages;
```

## Summary

`REGEXP_SUBSTR()` (MySQL 8.0+) extracts the first (or Nth) substring matching a regular expression from a string, returning `NULL` when no match exists. It is powerful for mining structured values from free-text columns like log messages, notes, and user-submitted content. Use the `occurrence` parameter to iterate through multiple matches and `match_type` flags to control case sensitivity.
