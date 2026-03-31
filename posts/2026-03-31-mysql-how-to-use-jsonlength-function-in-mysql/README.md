# How to Use JSON_LENGTH() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Length, JSON, JSON Function, SQL

Description: Learn how to use MySQL's JSON_LENGTH() function to count the elements in a JSON array or the keys in a JSON object, with examples for filtering and reporting.

---

## Overview

`JSON_LENGTH()` returns the number of elements in a JSON document. For a JSON array it counts the array elements; for a JSON object it counts the number of keys; for a scalar value it returns 1. It is useful for validation, filtering, and reporting on JSON data stored in MySQL.

## Basic Syntax

```sql
JSON_LENGTH(json_doc[, path])
```

The optional `path` argument restricts the length calculation to a nested portion of the document. Returns `NULL` if `json_doc` or `path` is `NULL`, or if the path does not exist.

## Basic Examples

```sql
-- Length of a JSON array
SELECT JSON_LENGTH('[1, 2, 3, 4, 5]');                    -- 5
SELECT JSON_LENGTH('["a", "b", "c"]');                    -- 3
SELECT JSON_LENGTH('[]');                                  -- 0

-- Length of a JSON object (number of top-level keys)
SELECT JSON_LENGTH('{"name":"Alice","age":30,"city":"NYC"}'); -- 3
SELECT JSON_LENGTH('{}');                                  -- 0

-- Length of a scalar
SELECT JSON_LENGTH('"hello"');                             -- 1
SELECT JSON_LENGTH('42');                                  -- 1

-- Length at a specific path
SELECT JSON_LENGTH('{"tags":["mysql","sql","db"]}', '$.tags'); -- 3
SELECT JSON_LENGTH('{"user":{"name":"Alice","age":30}}', '$.user'); -- 2
```

## Filtering Rows by JSON Array Size

```sql
CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  images JSON,
  tags JSON
);

INSERT INTO products (name, images, tags) VALUES
('Laptop', '["img1.jpg","img2.jpg","img3.jpg"]', '["tech","electronics"]'),
('Phone',  '["img1.jpg"]', '["tech","mobile","gadget"]'),
('Book',   '[]', '["education"]');

-- Find products with more than 2 images
SELECT id, name, JSON_LENGTH(images) AS image_count
FROM products
WHERE JSON_LENGTH(images) > 2;

-- Find products with no images
SELECT id, name
FROM products
WHERE JSON_LENGTH(images) = 0;

-- Find products with exactly 3 tags
SELECT id, name
FROM products
WHERE JSON_LENGTH(tags) = 3;
```

## Counting Keys in JSON Objects

```sql
CREATE TABLE user_settings (
  user_id INT PRIMARY KEY,
  preferences JSON
);

INSERT INTO user_settings VALUES
(1, '{"theme":"dark","lang":"en","font_size":14,"beta":true}'),
(2, '{"theme":"light"}'),
(3, '{"theme":"dark","lang":"fr","notifications":false}');

-- Users with more than 2 preference keys
SELECT user_id, JSON_LENGTH(preferences) AS pref_count
FROM user_settings
WHERE JSON_LENGTH(preferences) > 2;
```

## Using JSON_LENGTH() in Reports

```sql
-- Distribution of tag counts across products
SELECT
  JSON_LENGTH(tags) AS tag_count,
  COUNT(*) AS product_count
FROM products
GROUP BY JSON_LENGTH(tags)
ORDER BY tag_count;

-- Average number of tags per product
SELECT AVG(JSON_LENGTH(tags)) AS avg_tags
FROM products;
```

## JSON_LENGTH() with Nested Arrays

```sql
-- Count items in a nested array
SELECT JSON_LENGTH(
  '{"order":{"items":["item1","item2","item3"]}}',
  '$.order.items'
);  -- 3

-- Find orders with more than 5 items
SELECT order_id, JSON_LENGTH(order_data, '$.items') AS item_count
FROM orders
WHERE JSON_LENGTH(order_data, '$.items') > 5;
```

## Comparing JSON_LENGTH() and JSON_DEPTH()

```sql
-- JSON_LENGTH: number of elements at the top level (or at a path)
SELECT JSON_LENGTH('[[1,2],[3,4]]');     -- 2  (two sub-arrays)

-- JSON_DEPTH: maximum nesting depth of the entire document
SELECT JSON_DEPTH('[[1,2],[3,4]]');      -- 3  (array -> array -> scalar)
```

## Summary

`JSON_LENGTH()` counts the elements of a JSON array or the keys of a JSON object. Use the optional `path` argument to target nested structures. It is ideal for filtering rows by collection size, reporting on JSON schema completeness, and validating that arrays have the expected number of elements. Combine it with `JSON_DEPTH()` for a fuller picture of JSON document structure.
