# How to Use JSON_MERGE_PATCH() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Merge Patch, Json, Json Functions, Sql

Description: Learn how to use MySQL's JSON_MERGE_PATCH() function to merge JSON objects using RFC 7396 patch semantics, where later values override earlier ones.

---

## Overview

`JSON_MERGE_PATCH()` merges two or more JSON documents using RFC 7396 merge patch semantics. When a key exists in both documents, the value from the later document wins. When the patch value is JSON `null`, the key is removed from the result. This makes it the standard function for applying partial updates to JSON documents.

## Basic Syntax

```sql
JSON_MERGE_PATCH(json_doc, patch [, patch ...])
```

Multiple patch arguments are applied from left to right.

## Basic Examples

```sql
-- Simple merge: later values override earlier ones
SELECT JSON_MERGE_PATCH(
  '{"name":"Alice","age":25}',
  '{"age":30,"city":"NYC"}'
);
-- Result: {"name": "Alice", "age": 30, "city": "NYC"}

-- Removing a key with JSON null
SELECT JSON_MERGE_PATCH(
  '{"name":"Alice","age":25,"city":"LA"}',
  '{"city":null}'
);
-- Result: {"name": "Alice", "age": 25}

-- Adding a new key
SELECT JSON_MERGE_PATCH(
  '{"name":"Alice"}',
  '{"email":"alice@example.com"}'
);
-- Result: {"name": "Alice", "email": "alice@example.com"}

-- Multiple patches applied in order
SELECT JSON_MERGE_PATCH(
  '{"a":1}',
  '{"b":2}',
  '{"c":3}'
);
-- Result: {"a": 1, "b": 2, "c": 3}
```

## Nested Object Merging

```sql
-- Nested objects are merged recursively
SELECT JSON_MERGE_PATCH(
  '{"user":{"name":"Alice","age":25}}',
  '{"user":{"age":30,"city":"NYC"}}'
);
-- Result: {"user": {"name": "Alice", "age": 30, "city": "NYC"}}
```

## Arrays: Patch Overwrites, Not Appends

```sql
-- Arrays are replaced entirely (not merged)
SELECT JSON_MERGE_PATCH(
  '{"tags":["sql","mysql"]}',
  '{"tags":["database"]}'
);
-- Result: {"tags": ["database"]}  -- entire array replaced

-- Use JSON_ARRAY_APPEND instead if you want to add to an array
```

## Applying Patches to Stored JSON Columns

```sql
CREATE TABLE user_profiles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT,
  profile JSON
);

INSERT INTO user_profiles (user_id, profile) VALUES
(1, '{"name":"Alice","age":25,"city":"LA","newsletter":true}'),
(2, '{"name":"Bob","age":32,"city":"NYC"}');

-- Apply a patch to update age and remove newsletter
UPDATE user_profiles
SET profile = JSON_MERGE_PATCH(profile, '{"age":26,"newsletter":null}')
WHERE user_id = 1;

SELECT profile FROM user_profiles WHERE user_id = 1;
-- {"name": "Alice", "age": 26, "city": "LA"}
```

## Bulk Patch Application

```sql
-- Add a 'verified' field to all profiles that don't have it
UPDATE user_profiles
SET profile = JSON_MERGE_PATCH(profile, '{"verified":false}')
WHERE JSON_EXTRACT(profile, '$.verified') IS NULL;
```

## JSON_MERGE_PATCH() vs JSON_MERGE_PRESERVE()

```sql
SET @a = '{"tags":["sql"],"score":10}';
SET @b = '{"tags":["mysql"],"level":"beginner"}';

-- JSON_MERGE_PATCH: arrays are REPLACED, scalars are REPLACED
SELECT JSON_MERGE_PATCH(@a, @b);
-- {"tags": ["mysql"], "score": 10, "level": "beginner"}

-- JSON_MERGE_PRESERVE: arrays are MERGED (concatenated), scalars conflict creates array
SELECT JSON_MERGE_PRESERVE(@a, @b);
-- {"tags": ["sql", "mysql"], "score": 10, "level": "beginner"}
```

## Practical Example: REST API-Style PATCH Operation

```sql
-- Simulate a PATCH /users/1 request body
SET @patch_body = '{"email":"alice@new.com","age":null}';

UPDATE users
SET profile = JSON_MERGE_PATCH(profile, @patch_body)
WHERE id = 1;
```

This pattern mirrors how HTTP PATCH endpoints work: provide only the fields to change, set to `null` to delete.

## Summary

`JSON_MERGE_PATCH()` merges JSON objects using RFC 7396 semantics where later values override earlier ones and `null` removes keys. It is the correct function for applying partial updates to JSON documents because it handles nested objects recursively. Remember that arrays are replaced entirely - use `JSON_ARRAY_APPEND()` if you need to add elements to an existing array. Use `JSON_MERGE_PRESERVE()` when you want arrays to be concatenated instead.
