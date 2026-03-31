# How to Use JSON_VALID() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, JSON, Database

Description: Learn how to use MySQL JSON_VALID() to check whether a string is valid JSON before parsing, and how to use it in CHECK constraints to enforce column integrity.

---

## What JSON_VALID() Does

`JSON_VALID()` tests whether a value is valid JSON. It returns:

- `1` if the value is valid JSON (object, array, string, number, boolean, or null)
- `0` if the value is not valid JSON
- `NULL` if the argument is SQL `NULL`

```mermaid
flowchart TD
    A[Input value] --> B[JSON_VALID]
    B --> C{Valid JSON?}
    C -->|Yes| D[Returns 1]
    C -->|No| E[Returns 0]
    C -->|NULL input| F[Returns NULL]

    subgraph "Valid JSON examples"
    G["'{}' -> 1"]
    H["'[1,2,3]' -> 1"]
    I["'\"hello\"' -> 1"]
    J["'42' -> 1"]
    K["'true' -> 1"]
    end

    subgraph "Invalid JSON examples"
    L["'{a: 1}' -> 0  (unquoted key)"]
    M["'hello' -> 0   (bare word)"]
    N["'[1,2,]' -> 0  (trailing comma)"]
    end
```

## Syntax

```sql
JSON_VALID(val)
```

## Basic Examples

```sql
SELECT
    JSON_VALID('{}')            AS valid_object,
    JSON_VALID('[]')            AS valid_array,
    JSON_VALID('"hello"')       AS valid_string,
    JSON_VALID('42')            AS valid_number,
    JSON_VALID('true')          AS valid_bool,
    JSON_VALID('null')          AS valid_null,
    JSON_VALID('{a: 1}')        AS unquoted_key,
    JSON_VALID('hello')         AS bare_word,
    JSON_VALID('[1, 2,]')       AS trailing_comma,
    JSON_VALID('')               AS empty_string,
    JSON_VALID(NULL)             AS sql_null;
```

```text
+--------------+-------------+--------------+--------------+------------+------------+--------------+-----------+----------------+--------------+----------+
| valid_object | valid_array | valid_string | valid_number | valid_bool | valid_null | unquoted_key | bare_word | trailing_comma | empty_string | sql_null |
+--------------+-------------+--------------+--------------+------------+------------+--------------+-----------+----------------+--------------+----------+
|            1 |           1 |            1 |            1 |          1 |          1 |            0 |         0 |              0 |            0 |     NULL |
+--------------+-------------+--------------+--------------+------------+------------+--------------+-----------+----------------+--------------+----------+
```

## Enforcing Valid JSON with a CHECK Constraint

MySQL 8.0.16+ supports `CHECK` constraints. Use `JSON_VALID()` to ensure a `TEXT` or `VARCHAR` column only contains valid JSON:

```sql
CREATE TABLE events (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(50),
    payload    TEXT,
    CONSTRAINT chk_payload_json CHECK (JSON_VALID(payload))
);

-- This succeeds
INSERT INTO events (event_type, payload)
VALUES ('page_view', '{"url": "/home", "user_id": 42}');

-- This fails
INSERT INTO events (event_type, payload)
VALUES ('click', '{url: "/button"}');
-- ERROR 3819: Check constraint 'chk_payload_json' is violated.
```

Note: `JSON` column type validates automatically at insert time. Use `JSON_VALID()` in `CHECK` constraints when the column type is `TEXT` or `VARCHAR` and you need manual validation.

## Setup: Sample Table with TEXT Payload

```sql
CREATE TABLE raw_imports (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    source     VARCHAR(50),
    raw_data   TEXT,
    imported_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO raw_imports (source, raw_data) VALUES
('api_feed',   '{"event": "login", "user": 101}'),
('legacy_app', '{event: "click", user: 202}'),     -- invalid JSON
('webhook',    '{"event": "purchase", "amount": 49.99}'),
('file_import','not json at all'),                  -- invalid
('api_feed',   '{"event": "logout", "user": 101}');
```

## Finding Invalid JSON Rows

```sql
SELECT id, source, raw_data
FROM raw_imports
WHERE JSON_VALID(raw_data) = 0;
```

```text
+----+------------+----------------------------+
| id | source     | raw_data                   |
+----+------------+----------------------------+
|  2 | legacy_app | {event: "click", user: 202}|
|  4 | file_import| not json at all            |
+----+------------+----------------------------+
```

## Filtering and Processing Only Valid Rows

```sql
SELECT
    id,
    source,
    raw_data ->> '$.event' AS event_name
FROM raw_imports
WHERE JSON_VALID(raw_data) = 1;
```

Applying JSON path operators to invalid JSON would raise an error. Always guard with `JSON_VALID()` when the column type is `TEXT`.

## Audit Report: Valid vs Invalid Counts

```sql
SELECT
    source,
    SUM(JSON_VALID(raw_data))            AS valid_count,
    SUM(1 - JSON_VALID(raw_data))        AS invalid_count,
    COUNT(*)                             AS total
FROM raw_imports
GROUP BY source
ORDER BY invalid_count DESC;
```

## JSON_VALID() vs Using a JSON Column Type

```sql
-- Native JSON column: MySQL validates at insert time automatically
CREATE TABLE validated_events (
    id      INT AUTO_INCREMENT PRIMARY KEY,
    payload JSON NOT NULL
);

-- No need for JSON_VALID() check - invalid JSON is rejected at write time
INSERT INTO validated_events (payload) VALUES ('{invalid}');
-- ERROR 3140: Invalid JSON text: ...
```

Use a `JSON` column type when the data is always JSON. Use `JSON_VALID()` with `TEXT` columns when you need to store mixed content or handle legacy data.

## NULL Handling

```sql
SELECT JSON_VALID(NULL);   -- NULL (SQL null in, SQL null out)
SELECT JSON_VALID('null'); -- 1   (JSON null literal is valid JSON)
```

## Summary

`JSON_VALID()` returns 1 for well-formed JSON, 0 for invalid JSON, and NULL for a SQL NULL argument. It is primarily useful when working with `TEXT` or `VARCHAR` columns that may contain JSON, allowing you to filter valid rows before applying JSON path operations and to enforce JSON integrity via `CHECK` constraints. When a column is declared as `JSON` type, MySQL validates automatically and `JSON_VALID()` is not needed for data protection.
