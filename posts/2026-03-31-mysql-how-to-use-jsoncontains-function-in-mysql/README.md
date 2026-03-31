# How to Use JSON_CONTAINS() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Contains, Json, Json Functions, Sql

Description: Learn how to use MySQL's JSON_CONTAINS() function to check if a JSON document contains a specific value or sub-document at an optional path.

---

## Overview

`JSON_CONTAINS()` checks whether a target JSON document contains a specified candidate value. It returns 1 if the candidate is found, 0 if not, and `NULL` if either argument is `NULL`. You can optionally specify a JSON path to scope the search to a subset of the document.

## Basic Syntax

```sql
JSON_CONTAINS(target, candidate[, path])
```

- `target` - the JSON document to search
- `candidate` - the JSON value to look for (must be valid JSON)
- `path` - optional path within `target` to restrict the search

## Basic Examples

```sql
-- Check if a JSON object contains a key-value pair
SELECT JSON_CONTAINS('{"name":"Alice","age":30}', '{"name":"Alice"}');  -- 1
SELECT JSON_CONTAINS('{"name":"Bob","age":30}', '{"name":"Alice"}');    -- 0

-- Check if an array contains a value
SELECT JSON_CONTAINS('[1,2,3,4]', '3');     -- 1
SELECT JSON_CONTAINS('[1,2,3,4]', '5');     -- 0

-- Check with a path
SELECT JSON_CONTAINS('{"tags":["mysql","sql","database"]}', '"mysql"', '$.tags');  -- 1
SELECT JSON_CONTAINS('{"tags":["mysql","sql"]}', '"postgres"', '$.tags');           -- 0

-- Check for a nested object
SELECT JSON_CONTAINS(
  '{"user":{"name":"Alice","role":"admin"}}',
  '{"role":"admin"}',
  '$.user'
);  -- 1
```

## Filtering Rows by JSON Content

```sql
CREATE TABLE articles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200),
  tags JSON
);

INSERT INTO articles (title, tags) VALUES
('MySQL JSON Functions', '["mysql","sql","database"]'),
('PostgreSQL Arrays',   '["postgresql","sql","arrays"]'),
('Redis Caching',       '["redis","caching","nosql"]');

-- Find articles tagged with "sql"
SELECT id, title
FROM articles
WHERE JSON_CONTAINS(tags, '"sql"');

-- Find articles tagged with both "mysql" AND "sql"
SELECT id, title
FROM articles
WHERE JSON_CONTAINS(tags, '"mysql"') AND JSON_CONTAINS(tags, '"sql"');
```

## Checking Array Membership

```sql
-- Does the permissions array contain "write"?
SELECT user_id, name
FROM users
WHERE JSON_CONTAINS(permissions, '"write"');

-- Find users with admin role in their roles array
SELECT * FROM users
WHERE JSON_CONTAINS(roles, '"admin"');
```

## Using JSON_CONTAINS() with Indexes

For better performance on large tables, use a multi-valued index (MySQL 8.0+):

```sql
CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  categories JSON
);

-- Create a multi-valued index on the JSON array
ALTER TABLE products
  ADD INDEX idx_categories ((CAST(categories AS UNSIGNED ARRAY)));

-- Query benefits from the index
SELECT * FROM products
WHERE JSON_CONTAINS(categories, '5');  -- category id 5
```

Alternatively, use a generated column index:

```sql
ALTER TABLE articles
  ADD COLUMN has_mysql TINYINT AS (JSON_CONTAINS(tags, '"mysql"')) STORED,
  ADD INDEX idx_has_mysql (has_mysql);
```

## JSON_CONTAINS() vs JSON_SEARCH()

```sql
-- JSON_CONTAINS returns 1/0 (does document contain this value?)
SELECT JSON_CONTAINS('["a","b","c"]', '"b"');  -- 1

-- JSON_SEARCH returns the path where the value was found
SELECT JSON_SEARCH('["a","b","c"]', 'one', 'b');  -- "$[1]"
```

Use `JSON_CONTAINS()` for boolean existence checks; use `JSON_SEARCH()` when you need the path location.

## Practical Example: Feature Flags

```sql
CREATE TABLE feature_flags (
  user_id INT PRIMARY KEY,
  enabled_features JSON
);

INSERT INTO feature_flags VALUES
(1, '["dark_mode","beta_editor","analytics"]'),
(2, '["analytics"]'),
(3, '["dark_mode","beta_editor"]');

-- Find users who have the beta_editor feature enabled
SELECT user_id
FROM feature_flags
WHERE JSON_CONTAINS(enabled_features, '"beta_editor"');
-- Returns user_id: 1, 3
```

## Summary

`JSON_CONTAINS()` is the standard way to check whether a JSON document or array contains a specific value or sub-document. It returns 1 or 0, making it natural in `WHERE` clauses. Combine it with multi-valued indexes or generated column indexes in MySQL 8.0+ to keep queries fast on large JSON datasets.
