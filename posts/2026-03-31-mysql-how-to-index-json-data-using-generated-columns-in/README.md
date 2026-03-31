# How to Index JSON Data Using Generated Columns in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Json, Generated Columns, Indexing, Query Optimization

Description: Learn how to index JSON column fields in MySQL using generated columns to enable fast filtering and lookups on semi-structured data.

---

## Overview

MySQL does not support direct indexes on JSON columns. However, you can create a generated column that extracts a specific path from a JSON document, then index that generated column like any ordinary column. This approach gives you index-backed filtering on JSON fields without changing how you insert or update the JSON document.

Generated columns can be:
- **VIRTUAL** - computed on-the-fly, no extra storage cost
- **STORED** - persisted on disk, faster to read but uses more space

## Basic Syntax

```sql
-- Add a generated column from a JSON path
ALTER TABLE table_name
  ADD COLUMN col_name col_type
    GENERATED ALWAYS AS (json_col ->> '$.path') [VIRTUAL | STORED];

-- Then create an index on the generated column
CREATE INDEX idx_name ON table_name (col_name);
```

## Step-by-Step Example

Start with a products table that has a JSON `attributes` column:

```sql
CREATE TABLE products (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  name       VARCHAR(100) NOT NULL,
  price      DECIMAL(10, 2),
  attributes JSON
);

INSERT INTO products (name, price, attributes) VALUES
  ('Keyboard', 49.99, '{"color": "black",  "brand": "Logitech", "wireless": true}'),
  ('Mouse',    29.99, '{"color": "white",  "brand": "Razer",    "wireless": true}'),
  ('Monitor',  299.0, '{"color": "silver", "brand": "Dell",     "wireless": false}'),
  ('Webcam',   79.99, '{"color": "black",  "brand": "Logitech", "wireless": false}');
```

### Step 1: Add a Generated Column

```sql
ALTER TABLE products
  ADD COLUMN brand VARCHAR(50)
    GENERATED ALWAYS AS (attributes ->> '$.brand') VIRTUAL;
```

### Step 2: Create an Index on It

```sql
CREATE INDEX idx_products_brand ON products (brand);
```

### Step 3: Query Using the Generated Column

```sql
-- MySQL optimizer will use idx_products_brand
SELECT id, name, price
FROM products
WHERE brand = 'Logitech';
```

You can also query through the JSON expression directly and MySQL will still use the index:

```sql
-- This also uses the index (MySQL matches the expression)
SELECT id, name, price
FROM products
WHERE attributes ->> '$.brand' = 'Logitech';
```

## Verifying Index Usage with EXPLAIN

```sql
EXPLAIN SELECT id, name FROM products WHERE brand = 'Logitech'\G
```

Expected output includes `key: idx_products_brand`, confirming the index is used.

```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: products
         type: ref
possible_keys: idx_products_brand
          key: idx_products_brand
      key_len: 203
          ref: const
         rows: 2
     filtered: 100.00
        Extra: NULL
```

## Indexing a Numeric JSON Field

When the JSON field is a number, cast it to the appropriate type in the generated column definition:

```sql
ALTER TABLE products
  ADD COLUMN rating DECIMAL(3, 2)
    GENERATED ALWAYS AS (CAST(attributes ->> '$.rating' AS DECIMAL(3, 2))) VIRTUAL;

CREATE INDEX idx_products_rating ON products (rating);

-- Range query with index support
SELECT name, rating FROM products WHERE rating >= 4.5;
```

## Composite Index on Multiple JSON Fields

You can create a composite index across multiple generated columns:

```sql
ALTER TABLE products
  ADD COLUMN color VARCHAR(30)
    GENERATED ALWAYS AS (attributes ->> '$.color') VIRTUAL;

CREATE INDEX idx_brand_color ON products (brand, color);

-- Benefits from composite index
SELECT name FROM products WHERE brand = 'Logitech' AND color = 'black';
```

## STORED vs VIRTUAL Generated Columns

```sql
-- VIRTUAL: no extra disk space, column value computed on read
ADD COLUMN brand VARCHAR(50)
  GENERATED ALWAYS AS (attributes ->> '$.brand') VIRTUAL;

-- STORED: value written to disk at insert/update, faster SELECT
ADD COLUMN brand VARCHAR(50)
  GENERATED ALWAYS AS (attributes ->> '$.brand') STORED;
```

For InnoDB tables, you can index both VIRTUAL and STORED generated columns. Use STORED when the JSON expression is expensive to compute and you have write-tolerant workloads.

## Unique Index on a JSON Field

Enforce uniqueness on a JSON attribute:

```sql
ALTER TABLE products
  ADD COLUMN sku VARCHAR(50)
    GENERATED ALWAYS AS (attributes ->> '$.sku') STORED;

ALTER TABLE products
  ADD UNIQUE INDEX uq_products_sku (sku);
```

## Practical Example: User Preferences Table

```sql
CREATE TABLE user_preferences (
  user_id INT PRIMARY KEY,
  prefs   JSON
);

-- Index the theme preference for dashboard filtering
ALTER TABLE user_preferences
  ADD COLUMN theme VARCHAR(20)
    GENERATED ALWAYS AS (prefs ->> '$.theme') VIRTUAL;

CREATE INDEX idx_prefs_theme ON user_preferences (theme);

-- Fast lookup: all users with dark theme
SELECT user_id FROM user_preferences WHERE theme = 'dark';
```

## Limitations

```sql
-- You cannot index a JSON array element range directly
-- This does NOT use an index
WHERE JSON_CONTAINS(attributes->'$.tags', '"sale"')

-- For array membership, use FULLTEXT on a STORED generated column
-- or normalize the data into a separate junction table
```

Also note that generated column expressions must be deterministic and cannot reference other generated columns.

## Summary

MySQL does not support indexes directly on JSON columns, but you can extract any JSON path into a generated column and index it normally. Virtual generated columns cost no extra storage and still support indexes in InnoDB. Combine this technique with `EXPLAIN` to confirm index usage, and prefer stored generated columns when the JSON expression is complex and read performance is a priority.
