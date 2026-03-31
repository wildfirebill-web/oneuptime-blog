# How to Update Nested JSON Values in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, Update

Description: Learn how to update nested JSON values in MySQL using JSON_SET(), JSON_REPLACE(), and JSON_INSERT() with path expressions for precise document modifications.

---

## JSON Update Functions Overview

MySQL provides three functions for modifying existing JSON documents:

| Function | Creates new paths | Replaces existing | Summary |
|---|---|---|---|
| `JSON_SET()` | Yes | Yes | Upsert - insert or replace |
| `JSON_REPLACE()` | No | Yes | Replace only existing paths |
| `JSON_INSERT()` | Yes | No | Insert only new paths |

All three accept a JSON document, then pairs of path and value arguments.

## JSON_SET() - Upsert Behavior

`JSON_SET()` is the most commonly used update function. It sets a value at a path regardless of whether the path already exists:

```sql
SET @doc = '{"user": {"name": "Alice", "age": 30, "address": {"city": "NYC"}}}';

-- Update an existing nested field
SELECT JSON_SET(@doc, '$.user.age', 31);
-- Result: {"user": {"address": {"city": "NYC"}, "age": 31, "name": "Alice"}}

-- Add a new nested field
SELECT JSON_SET(@doc, '$.user.address.zip', '10001');
-- Result adds "zip" under address

-- Update multiple paths at once
SELECT JSON_SET(@doc,
  '$.user.age', 31,
  '$.user.address.zip', '10001',
  '$.user.active', TRUE
);
```

## JSON_REPLACE() - Replace Only

`JSON_REPLACE()` only modifies paths that already exist in the document. New paths are silently ignored:

```sql
-- This works - 'age' exists
SELECT JSON_REPLACE(@doc, '$.user.age', 35);

-- This does nothing - 'phone' does not exist
SELECT JSON_REPLACE(@doc, '$.user.phone', '555-1234');
-- Result: original document unchanged
```

Use `JSON_REPLACE()` when you want to guarantee you are only updating existing data, not adding new fields.

## JSON_INSERT() - Insert Only

`JSON_INSERT()` adds new paths but does not overwrite existing ones:

```sql
-- 'phone' does not exist - inserted
SELECT JSON_INSERT(@doc, '$.user.phone', '555-1234');

-- 'age' exists - not modified
SELECT JSON_INSERT(@doc, '$.user.age', 99);
-- Result: age stays at 30
```

## Updating JSON Columns in Tables

```sql
CREATE TABLE users (id INT PRIMARY KEY, profile JSON);
INSERT INTO users VALUES
  (1, '{"name": "Alice", "prefs": {"theme": "light", "lang": "en"}}'),
  (2, '{"name": "Bob", "prefs": {"theme": "dark", "lang": "fr"}}');

-- Update a nested preference for a specific user
UPDATE users
SET profile = JSON_SET(profile, '$.prefs.theme', 'dark')
WHERE id = 1;

-- Add a new nested field for all users
UPDATE users
SET profile = JSON_SET(profile, '$.prefs.notifications', TRUE);
```

## Updating Array Elements

Use index paths to update specific array positions:

```sql
UPDATE users
SET profile = JSON_SET(profile, '$.roles[0]', 'superadmin')
WHERE id = 1;
```

Append to an array:

```sql
UPDATE users
SET profile = JSON_ARRAY_APPEND(profile, '$.roles', 'editor')
WHERE id = 1;
```

## Conditional Updates with JSON_MERGE_PATCH()

For merging partial updates (overwriting only specified keys), use `JSON_MERGE_PATCH()`:

```sql
UPDATE users
SET profile = JSON_MERGE_PATCH(
  profile,
  JSON_OBJECT('prefs', JSON_OBJECT('theme', 'system', 'font_size', 14))
)
WHERE id = 1;
```

This merges the patch into the existing document, leaving unspecified keys intact.

## Removing Nested Fields

```sql
UPDATE users
SET profile = JSON_REMOVE(profile, '$.prefs.lang')
WHERE id = 1;
```

## Summary

Use `JSON_SET()` for upsert behavior when updating nested JSON in MySQL, `JSON_REPLACE()` when you only want to modify existing paths, and `JSON_INSERT()` when adding new fields without overwriting. For bulk partial updates, `JSON_MERGE_PATCH()` is the most powerful option.
