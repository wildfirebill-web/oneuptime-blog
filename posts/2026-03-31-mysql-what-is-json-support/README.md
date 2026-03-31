# What Is JSON Support in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, Data Type, Document Store, NoSQL

Description: MySQL JSON support includes a native JSON data type, path expressions for querying nested data, and a rich set of built-in functions for creating and manipulating JSON documents.

---

## Overview

MySQL introduced native JSON support in version 5.7.8. The `JSON` data type stores JSON documents in a binary format that is more efficient than storing them as plain text in a `VARCHAR` or `TEXT` column. MySQL validates JSON on insert, stores it in an optimized binary representation, and provides a comprehensive set of functions for extracting, modifying, and querying JSON values. MySQL 8.0 added multi-valued indexes on JSON arrays, making JSON queries significantly faster.

## The JSON Data Type

```sql
CREATE TABLE product_catalog (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  attributes JSON NOT NULL
);

INSERT INTO product_catalog (name, attributes) VALUES
('T-Shirt', '{"color": "blue", "sizes": ["S", "M", "L"], "weight_kg": 0.2}'),
('Laptop',  '{"brand": "Acme", "ram_gb": 16, "storage": {"type": "SSD", "gb": 512}}');
```

MySQL validates the JSON on insert and rejects malformed documents.

## Extracting Values

Use `->` and `->>` operators (shorthand for `JSON_EXTRACT` and `JSON_UNQUOTE(JSON_EXTRACT)`):

```sql
-- Extract with quotes
SELECT attributes->'$.color' AS color FROM product_catalog WHERE id = 1;
-- Returns "blue"

-- Extract without quotes (unquoted string)
SELECT attributes->>'$.color' AS color FROM product_catalog WHERE id = 1;
-- Returns blue

-- Nested path
SELECT attributes->>'$.storage.type' FROM product_catalog WHERE id = 2;
-- Returns SSD

-- Array element
SELECT attributes->>'$.sizes[0]' FROM product_catalog WHERE id = 1;
-- Returns S
```

## Filtering on JSON Values

```sql
-- Find products with 16GB RAM
SELECT * FROM product_catalog
WHERE attributes->>'$.ram_gb' = '16';

-- Use JSON_CONTAINS for array membership
SELECT * FROM product_catalog
WHERE JSON_CONTAINS(attributes->'$.sizes', '"M"');
```

## Modifying JSON

```sql
-- Set a value (creates or updates the path)
UPDATE product_catalog
SET attributes = JSON_SET(attributes, '$.color', 'red')
WHERE id = 1;

-- Remove a key
UPDATE product_catalog
SET attributes = JSON_REMOVE(attributes, '$.weight_kg')
WHERE id = 1;

-- Add an element to an array
UPDATE product_catalog
SET attributes = JSON_ARRAY_APPEND(attributes, '$.sizes', 'XL')
WHERE id = 1;
```

## Multi-Valued Indexes (MySQL 8.0)

Index values inside a JSON array for fast queries:

```sql
ALTER TABLE product_catalog
ADD INDEX idx_sizes ((CAST(attributes->'$.sizes' AS CHAR(10) ARRAY)));

-- Now this query uses the index
SELECT * FROM product_catalog
WHERE 'M' MEMBER OF (attributes->'$.sizes');
```

## Useful JSON Functions

```sql
-- Check if a JSON document is valid
SELECT JSON_VALID('{"key": "value"}');  -- 1
SELECT JSON_VALID('{invalid}');          -- 0

-- Get the type of a value
SELECT JSON_TYPE(attributes->'$.sizes');  -- ARRAY
SELECT JSON_TYPE(attributes->'$.ram_gb'); -- INTEGER

-- Pretty-print JSON
SELECT JSON_PRETTY(attributes) FROM product_catalog LIMIT 1;

-- Convert rows to JSON array
SELECT JSON_ARRAYAGG(name) FROM product_catalog;
-- Returns ["T-Shirt", "Laptop"]
```

## Summary

MySQL JSON support provides a validated, binary-optimized JSON data type with path expressions and extensive built-in functions for extraction, modification, and querying. It bridges relational and document models within a single database, making it practical to store semi-structured data alongside structured tables without a separate NoSQL store. Multi-valued indexes in MySQL 8.0 make JSON array searches index-efficient.
