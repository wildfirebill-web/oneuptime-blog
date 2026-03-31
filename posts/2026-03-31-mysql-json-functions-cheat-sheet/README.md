# MySQL JSON Functions Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON Function, Query, Cheat Sheet

Description: Quick reference for MySQL JSON functions including JSON_EXTRACT, JSON_SET, JSON_ARRAY, JSON_OBJECT, JSON_TABLE, and path expressions with practical examples.

---

## Storing and Retrieving JSON

```sql
CREATE TABLE configs (
  id   INT PRIMARY KEY,
  data JSON
);

INSERT INTO configs VALUES
  (1, '{"theme":"dark","lang":"en","features":["search","export"]}');

-- Two ways to extract
SELECT JSON_EXTRACT(data, '$.theme') FROM configs WHERE id = 1;
SELECT data->>'$.theme' FROM configs WHERE id = 1;  -- unquoted shorthand
```

## Path Syntax

```text
$           - root of the document
$.key       - top-level key
$.a.b       - nested key
$.arr[0]    - first element of an array
$.arr[*]    - all array elements
$**.key     - recursive descent
```

## Reading Values

```sql
SELECT data->'$.lang'           AS lang_quoted,    -- "en" (with quotes)
       data->>'$.lang'          AS lang_unquoted,   -- en
       JSON_UNQUOTE(data->'$.lang') AS lang_clean   -- en
FROM configs WHERE id = 1;

-- Array element
SELECT data->>'$.features[0]' FROM configs WHERE id = 1;  -- search
```

## Modifying JSON

```sql
-- JSON_SET: insert or update
UPDATE configs
SET data = JSON_SET(data, '$.theme', 'light', '$.version', 2)
WHERE id = 1;

-- JSON_INSERT: only inserts, does not overwrite
UPDATE configs SET data = JSON_INSERT(data, '$.newkey', 'value') WHERE id = 1;

-- JSON_REPLACE: only updates existing keys
UPDATE configs SET data = JSON_REPLACE(data, '$.theme', 'system') WHERE id = 1;

-- JSON_REMOVE: delete a key
UPDATE configs SET data = JSON_REMOVE(data, '$.lang') WHERE id = 1;
```

## Building JSON

```sql
SELECT JSON_OBJECT('name', 'Alice', 'age', 30);
-- {"name": "Alice", "age": 30}

SELECT JSON_ARRAY(1, 'two', true, NULL);
-- [1, "two", true, null]

SELECT JSON_ARRAYAGG(name) FROM employees WHERE dept_id = 1;
SELECT JSON_OBJECTAGG(id, name) FROM employees WHERE dept_id = 1;
```

## Checking and Searching

```sql
-- Check key existence
SELECT JSON_CONTAINS_PATH(data, 'one', '$.theme') FROM configs;

-- Check value existence
SELECT JSON_CONTAINS(data, '"dark"', '$.theme') FROM configs;

-- Search for a value, get its path
SELECT JSON_SEARCH(data, 'one', 'search', NULL, '$.features') FROM configs;
-- Returns "$.features[0]"

-- Check if valid JSON
SELECT JSON_VALID('{"a":1}');  -- 1
SELECT JSON_VALID('bad json'); -- 0
```

## Metadata

```sql
SELECT JSON_TYPE(data)               FROM configs;  -- OBJECT
SELECT JSON_TYPE(data->'$.features') FROM configs;  -- ARRAY
SELECT JSON_LENGTH(data)             FROM configs;  -- top-level key count
SELECT JSON_LENGTH(data->'$.features') FROM configs; -- array element count
SELECT JSON_DEPTH(data)              FROM configs;  -- nesting depth
SELECT JSON_KEYS(data)               FROM configs;  -- ["theme","lang","features"]
```

## JSON_TABLE (turn JSON into rows)

```sql
SELECT jt.*
FROM configs,
JSON_TABLE(
  data->'$.features',
  '$[*]' COLUMNS (
    feature VARCHAR(100) PATH '$'
  )
) AS jt
WHERE configs.id = 1;
```

## Indexing JSON (Generated Columns)

```sql
ALTER TABLE configs
  ADD COLUMN theme VARCHAR(50)
  GENERATED ALWAYS AS (data->>'$.theme') STORED;

CREATE INDEX idx_theme ON configs (theme);
```

## Summary

MySQL's JSON functions provide a rich API for querying and modifying JSON documents stored in JSON columns. Use `->` and `->>` for ergonomic path extraction, JSON_SET/JSON_REMOVE for in-place updates, and JSON_TABLE to pivot array data into relational rows. For query performance on JSON fields, add generated columns with indexes.
