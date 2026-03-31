# How to Use appendTrailingCharIfAbsent() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, Data Normalization, SQL

Description: Learn how appendTrailingCharIfAbsent() normalizes strings by appending a character only when it is not already present, ideal for URL paths and delimited data.

---

Data arriving from multiple sources rarely follows a consistent format. URL paths might sometimes end with a trailing slash and sometimes not. Log lines may or may not end with a newline. Before joining, grouping, or comparing such strings, you need to normalize them. ClickHouse's `appendTrailingCharIfAbsent()` function handles exactly this case: it appends a single character to a string only when the string does not already end with that character.

## Function Signature

```text
appendTrailingCharIfAbsent(str, char)
```

- `str` - the input string
- `char` - a single character to append if absent

The function returns a new string. If `str` already ends with `char`, it is returned unchanged.

## Basic Usage

```sql
-- Append a trailing slash to paths that lack one
SELECT
    appendTrailingCharIfAbsent('/api/v2/users', '/')   AS already_has_slash,
    appendTrailingCharIfAbsent('/api/v2/users/', '/')  AS added_slash;
```

```text
already_has_slash | added_slash
------------------|------------
/api/v2/users/    | /api/v2/users/
```

Both results are identical: when the slash is absent it gets appended, and when it is already present nothing changes.

## Normalizing URL Paths

Inconsistent trailing slashes are one of the most common causes of grouping mismatches in web analytics. Without normalization, `/dashboard` and `/dashboard/` appear as two separate paths in aggregations.

```sql
-- Normalize request paths before grouping
SELECT
    appendTrailingCharIfAbsent(request_path, '/') AS normalized_path,
    count() AS requests,
    uniq(user_id) AS unique_users
FROM http_logs
WHERE event_date = today()
GROUP BY normalized_path
ORDER BY requests DESC
LIMIT 20;
```

```sql
-- Compare grouped counts before and after normalization
SELECT
    request_path              AS raw_path,
    count()                   AS raw_count
FROM http_logs
WHERE event_date = today()
GROUP BY raw_path
HAVING raw_path IN ('/dashboard', '/dashboard/')
ORDER BY raw_path;
```

```sql
-- After normalization the two rows collapse into one
SELECT
    appendTrailingCharIfAbsent(request_path, '/') AS normalized_path,
    count() AS total_requests
FROM http_logs
WHERE event_date = today()
  AND request_path IN ('/dashboard', '/dashboard/')
GROUP BY normalized_path;
```

## Building Consistent Base URLs for Concatenation

When constructing full URLs by concatenating a base path and a relative path, you need the base to end with `/` to avoid double-slash or missing-slash issues.

```sql
-- Safely concatenate base URL with resource path
SELECT
    concat(
        appendTrailingCharIfAbsent(base_url, '/'),
        resource_path
    ) AS full_url
FROM (
    SELECT 'https://example.com/api'  AS base_url, 'users/42' AS resource_path UNION ALL
    SELECT 'https://example.com/api/', 'users/42'
);
```

```text
full_url
-----------------------------------------
https://example.com/api/users/42
https://example.com/api/users/42
```

Both input rows produce the same output regardless of whether the base already had a trailing slash.

## Ensuring Trailing Newlines in Text Fields

Some text processing pipelines expect each record to end with a newline character. `appendTrailingCharIfAbsent()` can enforce this invariant.

```sql
-- Ensure each exported log line ends with a newline
SELECT
    appendTrailingCharIfAbsent(log_line, char(10)) AS normalized_line
FROM export_queue
LIMIT 100;
```

`char(10)` is the ASCII newline character. The function appends it only when the line does not already end with one.

## Using It in a Materialized Column

If you frequently query normalized paths, you can store the normalized form directly in the table using a materialized column.

```sql
ALTER TABLE http_logs
ADD COLUMN normalized_path String
MATERIALIZED appendTrailingCharIfAbsent(request_path, '/');
```

After adding this column, all future inserts populate it automatically, and you can build indexes or projections on it without repeating the function call in every query.

## Applying to Multiple Characters with a Custom Separator

The function only works with a single character, but you can chain calls if you need two trailing characters.

```sql
-- Ensure a string ends with "\r\n" (carriage return + newline)
SELECT
    appendTrailingCharIfAbsent(
        appendTrailingCharIfAbsent(line, char(13)),
        char(10)
    ) AS crlf_terminated
FROM raw_lines
LIMIT 5;
```

Note that chaining works when you need each character appended independently; for multi-character suffixes you would use a `CASE` expression with `endsWith()`.

## Summary

`appendTrailingCharIfAbsent()` is a small but useful normalization function. Its primary strength is idempotency: applying it multiple times produces the same result, making it safe to use in ETL pipelines, materialized columns, and query-time normalization. Use it to standardize trailing slashes on URL paths, enforce newline terminators on text fields, and eliminate grouping mismatches caused by inconsistent string suffixes.
