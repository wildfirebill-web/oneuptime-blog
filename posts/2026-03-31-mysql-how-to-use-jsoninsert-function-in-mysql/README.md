# How to Use JSON_INSERT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Insert, Json, Json Functions, Sql

Description: Learn how to use MySQL's JSON_INSERT() function to add new keys to a JSON document without overwriting existing values.

---

## Overview

`JSON_INSERT()` inserts a new value into a JSON document at the specified path, but only if the path does not already exist. If the path already has a value, the original value is preserved. This non-destructive insert behavior makes it safe to use when you want to add defaults or new keys without risking overwriting existing data.

## Basic Syntax

```sql
JSON_INSERT(json_doc, path, val [, path, val] ...)
```

- `json_doc` - the source JSON document
- `path` - a JSON path expression (e.g., `'$.key'`)
- `val` - the value to insert if the path does not exist

Multiple path-value pairs can be provided in one call.

## Basic Examples

```sql
-- Insert a new key
SELECT JSON_INSERT('{"name":"Alice"}', '$.age', 30);
-- Result: {"name": "Alice", "age": 30}

-- Path already exists - value is NOT overwritten
SELECT JSON_INSERT('{"name":"Alice","age":25}', '$.age', 30);
-- Result: {"name": "Alice", "age": 25}  (unchanged)

-- Insert multiple keys
SELECT JSON_INSERT('{"name":"Alice"}', '$.age', 30, '$.city', 'NYC');
-- Result: {"name": "Alice", "age": 30, "city": "NYC"}

-- Insert into a nested path
SELECT JSON_INSERT('{"user":{"name":"Alice"}}', '$.user.age', 30);
-- Result: {"user": {"name": "Alice", "age": 30}}

-- Insert into an array by index (only if index doesn't exist)
SELECT JSON_INSERT('["a","b","c"]', '$[3]', 'd');
-- Result: ["a", "b", "c", "d"]
```

## Inserting into Stored JSON Columns

```sql
CREATE TABLE profiles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT,
  data JSON
);

INSERT INTO profiles (user_id, data) VALUES
(1, '{"name":"Alice","prefs":{}}');

-- Add a new field 'newsletter' only if it doesn't already exist
UPDATE profiles
SET data = JSON_INSERT(data, '$.prefs.newsletter', true)
WHERE user_id = 1;

SELECT data FROM profiles WHERE user_id = 1;
-- {"name": "Alice", "prefs": {"newsletter": true}}
```

## Difference Between JSON_INSERT(), JSON_SET(), and JSON_REPLACE()

```sql
-- Setup
SET @doc = '{"name":"Alice","age":25}';

-- JSON_INSERT: inserts only if path does NOT exist
SELECT JSON_INSERT(@doc, '$.age', 30, '$.city', 'NYC');
-- {"name": "Alice", "age": 25, "city": "NYC"}  (age unchanged, city added)

-- JSON_SET: inserts if not exists, replaces if exists
SELECT JSON_SET(@doc, '$.age', 30, '$.city', 'NYC');
-- {"name": "Alice", "age": 30, "city": "NYC"}  (age updated, city added)

-- JSON_REPLACE: replaces only if path EXISTS
SELECT JSON_REPLACE(@doc, '$.age', 30, '$.city', 'NYC');
-- {"name": "Alice", "age": 30}  (age updated, city NOT added)
```

## Adding Default Values to Existing Documents

```sql
-- Add default theme preference only if not already set
UPDATE user_settings
SET preferences = JSON_INSERT(
  COALESCE(preferences, '{}'),
  '$.theme', 'light',
  '$.language', 'en',
  '$.notifications', true
)
WHERE user_id IN (SELECT id FROM users WHERE created_at > '2024-01-01');
```

## Inserting Nested Structures

```sql
-- Insert a nested object at a new key
SELECT JSON_INSERT(
  '{"user":"Alice"}',
  '$.address',
  JSON_OBJECT('city', 'New York', 'zip', '10001')
);
-- {"user": "Alice", "address": {"city": "New York", "zip": "10001"}}
```

## Practical Example: Onboarding Defaults

```sql
-- Add default onboarding fields to all user profiles that lack them
UPDATE users
SET metadata = JSON_INSERT(
  COALESCE(metadata, '{}'),
  '$.onboarding_complete', false,
  '$.tutorial_step', 1,
  '$.welcome_shown', false
)
WHERE metadata IS NULL
   OR JSON_EXTRACT(metadata, '$.onboarding_complete') IS NULL;
```

## Summary

`JSON_INSERT()` safely adds new keys or array elements to a JSON document only when the specified path does not already exist, leaving existing values untouched. Use it when you want to provide defaults without overwriting current data. For unconditional updates, use `JSON_SET()`; to update only existing paths, use `JSON_REPLACE()`.
