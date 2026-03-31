# How to Use the JSON Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, Data Type, Database Design, NoSQL

Description: Learn how to store, query, and manipulate JSON data in MySQL using the native JSON data type and its built-in functions.

---

## Introduction to the JSON Data Type

MySQL 5.7.8 introduced a native `JSON` data type that stores JSON documents in an optimized binary format. It provides automatic validation, efficient access to individual elements, and a rich set of functions for querying and modifying JSON data.

```sql
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    attributes JSON,
    metadata JSON
);
```

## Inserting JSON Data

```sql
-- Insert a JSON object
INSERT INTO products (name, attributes) VALUES (
    'Laptop Pro',
    '{"brand": "TechCorp", "ram_gb": 16, "storage_gb": 512, "colors": ["silver", "black"]}'
);

-- Insert using JSON_OBJECT() function
INSERT INTO products (name, attributes) VALUES (
    'Gaming Mouse',
    JSON_OBJECT(
        'brand', 'GameGear',
        'dpi', 16000,
        'buttons', 7,
        'wireless', true
    )
);

-- Insert using JSON_ARRAY()
INSERT INTO products (name, attributes) VALUES (
    'Monitor Pack',
    JSON_OBJECT('sizes', JSON_ARRAY(24, 27, 32), 'resolution', '4K')
);
```

## Reading JSON Values with JSON_EXTRACT

Use `JSON_EXTRACT()` or the shorthand `->` operator to access JSON fields:

```sql
-- Using JSON_EXTRACT
SELECT name, JSON_EXTRACT(attributes, '$.brand') AS brand
FROM products;

-- Using -> shorthand (returns JSON type with quotes)
SELECT name, attributes->'$.brand' AS brand
FROM products;

-- Using ->> shorthand (returns unquoted string)
SELECT name, attributes->>'$.brand' AS brand
FROM products;

-- Access nested values
SELECT name, attributes->>'$.colors[0]' AS primary_color
FROM products;

-- Access array elements
SELECT name, JSON_EXTRACT(attributes, '$.colors[1]') AS second_color
FROM products;
```

## Filtering on JSON Values

```sql
-- Filter by a JSON field value
SELECT name FROM products
WHERE attributes->>'$.brand' = 'TechCorp';

-- Filter on numeric JSON field
SELECT name FROM products
WHERE JSON_EXTRACT(attributes, '$.ram_gb') >= 16;

-- Check if a key exists using JSON_CONTAINS_PATH
SELECT name FROM products
WHERE JSON_CONTAINS_PATH(attributes, 'one', '$.wireless') = 1;
```

## Modifying JSON Data

```sql
-- Update a specific key
UPDATE products
SET attributes = JSON_SET(attributes, '$.ram_gb', 32)
WHERE name = 'Laptop Pro';

-- Insert a new key (only if it does not exist)
UPDATE products
SET attributes = JSON_INSERT(attributes, '$.price', 999.99)
WHERE name = 'Laptop Pro';

-- Replace an existing key value
UPDATE products
SET attributes = JSON_REPLACE(attributes, '$.brand', 'TechCorp Inc')
WHERE name = 'Laptop Pro';

-- Remove a key
UPDATE products
SET attributes = JSON_REMOVE(attributes, '$.colors')
WHERE name = 'Laptop Pro';
```

## Generated Columns and Indexes on JSON

You cannot index a JSON column directly, but you can create generated columns from JSON paths and index those:

```sql
ALTER TABLE products
ADD COLUMN brand VARCHAR(100)
    GENERATED ALWAYS AS (attributes->>'$.brand') STORED,
ADD INDEX idx_brand (brand);

-- Now this query uses the index
SELECT name FROM products WHERE brand = 'TechCorp';
```

## Aggregating JSON Data

```sql
-- Build a JSON array from rows using JSON_ARRAYAGG
SELECT JSON_ARRAYAGG(name) AS product_names
FROM products;

-- Build a JSON object from rows using JSON_OBJECTAGG
SELECT JSON_OBJECTAGG(id, name) AS id_to_name
FROM products;

-- Pretty-print JSON output
SELECT name, JSON_PRETTY(attributes) AS formatted
FROM products
WHERE id = 1;
```

## Checking JSON Validity

```sql
-- JSON data type validates on insert, but you can also check with JSON_VALID
SELECT JSON_VALID('{"key": "value"}');  -- Returns 1
SELECT JSON_VALID('{bad json}');         -- Returns 0

-- Get the JSON type of a value
SELECT JSON_TYPE(attributes) FROM products LIMIT 1;  -- Returns 'OBJECT'
SELECT JSON_TYPE(attributes->'$.colors') FROM products LIMIT 1;  -- Returns 'ARRAY'
```

## Summary

MySQL's native JSON data type provides automatic validation, efficient binary storage, and a comprehensive function library for querying and modifying JSON documents. Use generated columns with indexes to optimize frequent lookups on specific JSON paths. While JSON columns offer flexibility for semi-structured data, consider normalizing frequently accessed fields into dedicated columns for best performance.
