# How to Validate JSON Data in MySQL with JSON_VALID()

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Valid, Json, Data Validation, Sql

Description: Learn how to use MySQL's JSON_VALID() function to check whether a string contains valid JSON, and how to apply it when importing or inserting data.

---

## Overview

`JSON_VALID()` is a MySQL function that returns 1 if the supplied string is a valid JSON document, or 0 if it is not. It is the first line of defense before parsing JSON strings with other JSON functions, and is particularly useful when processing data imported from external sources.

## Basic Syntax

```sql
JSON_VALID(val)
```

Returns `1` for valid JSON, `0` for invalid JSON, and `NULL` if the input is `NULL`.

## Basic Examples

```sql
-- Valid JSON object
SELECT JSON_VALID('{"name":"Alice","age":30}');  -- 1

-- Valid JSON array
SELECT JSON_VALID('[1, 2, 3]');                  -- 1

-- Valid JSON primitives
SELECT JSON_VALID('"hello"');                    -- 1
SELECT JSON_VALID('42');                         -- 1
SELECT JSON_VALID('true');                       -- 1
SELECT JSON_VALID('null');                       -- 1

-- Invalid JSON
SELECT JSON_VALID('{name: "Alice"}');            -- 0 (keys must be quoted)
SELECT JSON_VALID('{"name":"Alice"');            -- 0 (missing closing brace)
SELECT JSON_VALID('undefined');                  -- 0

-- NULL input
SELECT JSON_VALID(NULL);                         -- NULL
```

## Filtering Valid JSON Rows

```sql
-- Only select rows with valid JSON in a VARCHAR column
SELECT id, raw_json
FROM imports
WHERE JSON_VALID(raw_json) = 1;

-- Find rows with invalid JSON for cleanup
SELECT id, raw_json
FROM imports
WHERE JSON_VALID(raw_json) = 0 OR raw_json IS NULL;
```

## Enforcing JSON Validity with CHECK Constraints

In MySQL 8.0.16+, you can use `JSON_VALID()` in a `CHECK` constraint on a `VARCHAR` column:

```sql
CREATE TABLE user_settings (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT,
  settings VARCHAR(4000),
  CONSTRAINT chk_valid_json CHECK (JSON_VALID(settings))
);

-- This will succeed
INSERT INTO user_settings (user_id, settings)
VALUES (1, '{"theme":"dark","lang":"en"}');

-- This will fail the CHECK constraint
INSERT INTO user_settings (user_id, settings)
VALUES (2, '{invalid}');
```

## Validating Before Parsing

```sql
-- Safe extraction: validate before extracting keys
SELECT
  id,
  CASE WHEN JSON_VALID(payload)
    THEN JSON_UNQUOTE(JSON_EXTRACT(payload, '$.event_type'))
    ELSE 'INVALID'
  END AS event_type
FROM event_log;
```

## Bulk Validation During Data Import

```sql
-- Load from a staging table and validate
CREATE TABLE staging_events (
  id INT AUTO_INCREMENT PRIMARY KEY,
  raw_data TEXT
);

-- After loading, move valid rows to production table
INSERT INTO events (payload, imported_at)
SELECT raw_data, NOW()
FROM staging_events
WHERE JSON_VALID(raw_data) = 1;

-- Log invalid rows
INSERT INTO import_errors (raw_data, error_reason, logged_at)
SELECT raw_data, 'invalid JSON', NOW()
FROM staging_events
WHERE JSON_VALID(raw_data) = 0;
```

## Using JSON_VALID() with Generated Columns

```sql
-- Create a generated column that flags JSON validity
ALTER TABLE api_logs
ADD COLUMN is_valid_json TINYINT AS (JSON_VALID(request_body)) STORED;

-- Index the validity flag for fast filtering
ALTER TABLE api_logs ADD INDEX idx_valid_json (is_valid_json);

-- Query only valid logs
SELECT * FROM api_logs WHERE is_valid_json = 1;
```

## JSON_VALID() vs JSON Column Type

```sql
-- JSON type columns are always valid (MySQL enforces it on insert)
CREATE TABLE config (
  id INT PRIMARY KEY,
  settings JSON  -- always valid, validated on insert
);

-- VARCHAR columns need JSON_VALID() to check validity
CREATE TABLE raw_config (
  id INT PRIMARY KEY,
  settings VARCHAR(4000)  -- may contain invalid JSON
);
```

## Summary

`JSON_VALID()` returns 1 for valid JSON and 0 for invalid, making it the safest way to validate JSON strings in `VARCHAR` or `TEXT` columns before attempting extraction with `JSON_EXTRACT()` or other JSON functions. Use it in `WHERE` filters, `CHECK` constraints, or generated columns to enforce and quickly identify JSON validity in your data pipeline.
