# How to Use JSON_REPLACE() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Replace, JSON, JSON Function, SQL

Description: Learn how to use MySQL's JSON_REPLACE() function to update existing values in a JSON document without adding new keys.

---

## Overview

`JSON_REPLACE()` updates the value at an existing path in a JSON document. Unlike `JSON_SET()`, it will not add a new key if the path does not already exist. This makes `JSON_REPLACE()` the safe choice when you want to update known fields without accidentally creating new ones.

## Basic Syntax

```sql
JSON_REPLACE(json_doc, path, val [, path, val] ...)
```

Returns the modified JSON document. If the path does not exist, the document is returned unchanged for that path. Returns `NULL` if `json_doc` or any `path` is `NULL`.

## Basic Examples

```sql
-- Replace an existing key
SELECT JSON_REPLACE('{"name":"Alice","age":25}', '$.age', 30);
-- Result: {"name": "Alice", "age": 30}

-- Path does NOT exist - document unchanged
SELECT JSON_REPLACE('{"name":"Alice"}', '$.age', 30);
-- Result: {"name": "Alice"}  (age NOT added)

-- Replace multiple keys at once
SELECT JSON_REPLACE('{"name":"Alice","age":25,"city":"LA"}',
  '$.age', 30, '$.city', 'NYC');
-- Result: {"name": "Alice", "age": 30, "city": "NYC"}

-- Replace a nested value
SELECT JSON_REPLACE('{"user":{"name":"Alice","age":25}}', '$.user.age', 30);
-- Result: {"user": {"name": "Alice", "age": 30}}

-- Replace an array element
SELECT JSON_REPLACE('["a","b","c"]', '$[1]', 'B');
-- Result: ["a", "B", "c"]
```

## Updating JSON Columns in a Table

```sql
CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  attributes JSON
);

INSERT INTO products (name, attributes) VALUES
('Laptop', '{"brand":"Dell","ram":16,"storage":512}'),
('Phone', '{"brand":"Apple","ram":8,"storage":256}');

-- Update storage for all Dell products
UPDATE products
SET attributes = JSON_REPLACE(attributes, '$.storage', 1024)
WHERE JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.brand')) = 'Dell';

SELECT name, attributes FROM products;
```

## JSON_REPLACE() vs JSON_SET() vs JSON_INSERT()

```sql
SET @doc = '{"name":"Alice","age":25}';

-- JSON_REPLACE: updates existing, ignores missing paths
SELECT JSON_REPLACE(@doc, '$.age', 30, '$.city', 'NYC');
-- {"name": "Alice", "age": 30}  -- city is NOT added

-- JSON_SET: updates existing AND inserts missing
SELECT JSON_SET(@doc, '$.age', 30, '$.city', 'NYC');
-- {"name": "Alice", "age": 30, "city": "NYC"}  -- city IS added

-- JSON_INSERT: inserts missing, keeps existing
SELECT JSON_INSERT(@doc, '$.age', 30, '$.city', 'NYC');
-- {"name": "Alice", "age": 25, "city": "NYC"}  -- age unchanged, city added
```

## Replacing Array Elements by Index

```sql
-- Replace specific array elements
SELECT JSON_REPLACE('["red","green","blue"]', '$[0]', 'crimson', '$[2]', 'navy');
-- Result: ["crimson", "green", "navy"]

-- Update a nested array element
SELECT JSON_REPLACE('{"colors":["red","green"]}', '$.colors[0]', 'crimson');
-- Result: {"colors": ["crimson", "green"]}
```

## Practical Example: Updating User Preferences

```sql
-- Users table with JSON preferences
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  preferences JSON
);

INSERT INTO users (name, preferences) VALUES
('Alice', '{"theme":"light","lang":"en","notifications":true}'),
('Bob',   '{"theme":"dark","lang":"es","notifications":false}');

-- Alice switches to dark mode
UPDATE users
SET preferences = JSON_REPLACE(preferences, '$.theme', 'dark')
WHERE id = 1;

-- Bulk update: disable notifications (only for users who have the key)
UPDATE users
SET preferences = JSON_REPLACE(preferences, '$.notifications', false)
WHERE JSON_EXTRACT(preferences, '$.notifications') IS NOT NULL;
```

## Conditional Replace with Validation

```sql
-- Replace value only if current value matches expected value
UPDATE products
SET attributes = JSON_REPLACE(attributes, '$.status', 'discontinued')
WHERE id = 5
  AND JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.status')) = 'available';
```

## Summary

`JSON_REPLACE()` updates values at existing JSON paths and silently skips paths that do not exist, making it safe for updating known fields without schema drift. Use it when you need strict update-only semantics. Choose `JSON_SET()` when you want upsert behavior, and `JSON_INSERT()` when you want insert-only behavior that preserves existing values.
