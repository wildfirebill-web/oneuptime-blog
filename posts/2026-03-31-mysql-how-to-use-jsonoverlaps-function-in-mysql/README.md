# How to Use JSON_OVERLAPS() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Overlaps, JSON, JSON Function, SQL

Description: Learn how to use MySQL's JSON_OVERLAPS() function to check if two JSON documents share any common elements, introduced in MySQL 8.0.17.

---

## Overview

`JSON_OVERLAPS()` was introduced in MySQL 8.0.17 to check whether two JSON documents have any elements in common. It returns 1 if any element in the first document appears in the second, and 0 if there is no overlap. It is particularly useful for checking if two JSON arrays share any values without requiring a full containment check.

## Basic Syntax

```sql
JSON_OVERLAPS(json_doc1, json_doc2)
```

Returns 1 if the two documents overlap, 0 if they do not, and `NULL` if either argument is `NULL`. The comparison is symmetric - the order of arguments does not matter.

## Basic Examples

```sql
-- Arrays with common elements
SELECT JSON_OVERLAPS('[1,2,3]', '[3,4,5]');    -- 1  (3 is shared)
SELECT JSON_OVERLAPS('[1,2,3]', '[4,5,6]');    -- 0  (no overlap)

-- Objects with a common key-value pair
SELECT JSON_OVERLAPS('{"a":1,"b":2}', '{"b":2,"c":3}');  -- 1  (b:2 shared)
SELECT JSON_OVERLAPS('{"a":1,"b":2}', '{"b":3,"c":4}');  -- 0  (b has different values)

-- Scalar against scalar
SELECT JSON_OVERLAPS('5', '5');    -- 1
SELECT JSON_OVERLAPS('5', '6');    -- 0

-- Array vs scalar: checks if scalar is in the array
SELECT JSON_OVERLAPS('[1,2,3]', '2');  -- 1
SELECT JSON_OVERLAPS('[1,2,3]', '5');  -- 0

-- String values
SELECT JSON_OVERLAPS('["mysql","sql","database"]', '["postgresql","sql"]');  -- 1
```

## Filtering Rows with Overlapping JSON Arrays

```sql
CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  categories JSON
);

INSERT INTO products (name, categories) VALUES
('MySQL Book',     '["database","sql","mysql"]'),
('Redis Guide',    '["database","nosql","caching"]'),
('Python Basics',  '["programming","python"]'),
('SQL Cookbook',   '["database","sql"]');

-- Find products in the "database" or "sql" categories
SELECT id, name
FROM products
WHERE JSON_OVERLAPS(categories, '["database","sql"]');
-- Returns MySQL Book, Redis Guide, SQL Cookbook

-- Find products with any of the user's preferred categories
SET @user_prefs = '["python","caching"]';
SELECT id, name
FROM products
WHERE JSON_OVERLAPS(categories, @user_prefs);
-- Returns Redis Guide, Python Basics
```

## JSON_OVERLAPS() vs JSON_CONTAINS()

```sql
-- JSON_CONTAINS: checks if second is a SUBSET of first (full containment)
SELECT JSON_CONTAINS('[1,2,3,4]', '[2,3]');    -- 1 (both 2 and 3 must be in first)
SELECT JSON_CONTAINS('[1,2,3,4]', '[2,5]');    -- 0 (5 is not in first)

-- JSON_OVERLAPS: checks if ANY element is shared (partial match)
SELECT JSON_OVERLAPS('[1,2,3,4]', '[2,5]');    -- 1 (2 is shared, even though 5 is not)
SELECT JSON_OVERLAPS('[1,2,3,4]', '[5,6]');    -- 0 (no shared elements)
```

Use `JSON_OVERLAPS()` for "any of" logic and `JSON_CONTAINS()` for "all of" logic.

## Using JSON_OVERLAPS() with Multi-Valued Indexes

```sql
-- Multi-valued index for efficient JSON array queries (MySQL 8.0+)
CREATE TABLE articles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200),
  tags JSON
);

ALTER TABLE articles
  ADD INDEX idx_tags ((CAST(tags AS CHAR(100) ARRAY)));

-- JSON_OVERLAPS can benefit from the multi-valued index
SELECT * FROM articles
WHERE JSON_OVERLAPS(tags, '["mysql","database"]');
```

## Practical Example: Matching Users to Events

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  interests JSON
);

CREATE TABLE events (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200),
  topics JSON
);

INSERT INTO users (name, interests) VALUES
('Alice', '["mysql","database","python"]'),
('Bob',   '["redis","caching","docker"]');

INSERT INTO events (title, topics) VALUES
('MySQL Summit', '["mysql","sql","database"]'),
('DevOps Day',   '["docker","kubernetes","ci"]'),
('Cache Wars',   '["redis","caching","memcached"]');

-- Find events that match each user's interests
SELECT u.name, e.title
FROM users u
JOIN events e ON JSON_OVERLAPS(u.interests, e.topics)
ORDER BY u.name, e.title;
```

## Summary

`JSON_OVERLAPS()` checks if two JSON documents share any common elements and returns 1 or 0. It is symmetric and particularly useful for "any of" filtering on JSON arrays, matching users to tags or categories, and finding related records. Unlike `JSON_CONTAINS()`, which requires full containment, `JSON_OVERLAPS()` returns true as long as at least one element is shared. Use multi-valued indexes in MySQL 8.0+ to optimize queries using this function.
