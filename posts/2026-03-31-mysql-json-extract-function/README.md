# How to Use JSON_EXTRACT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, JSON, Database, Function

Description: Learn how to use MySQL JSON_EXTRACT() to retrieve values from JSON documents using JSONPath expressions, with examples for scalars, objects, and arrays.

---

## What JSON_EXTRACT() Does

`JSON_EXTRACT()` reads a value from a JSON column or expression using a JSONPath selector. It returns the extracted value in JSON encoding (preserving type information such as quotes for strings). If no match is found, it returns `NULL`.

```sql
SELECT JSON_EXTRACT('{"name": "Alice", "age": 30}', '$.name');
-- Result: "Alice"  (with quotes - JSON string)
```

The function signature is:

```sql
JSON_EXTRACT(json_doc, path [, path ...])
```

You can pass multiple paths; the function returns a JSON array of all matched values.

## JSONPath Basics

| Expression | Meaning |
|---|---|
| `$` | Root of the document |
| `$.key` | Key at the root level |
| `$.obj.key` | Nested key |
| `$.arr[0]` | First element of an array |
| `$.arr[*]` | All elements of an array |

## Basic Scalar Extraction

```sql
SET @doc = '{"product": "Widget", "price": 9.99, "in_stock": true}';

SELECT
  JSON_EXTRACT(@doc, '$.product')  AS product,
  JSON_EXTRACT(@doc, '$.price')    AS price,
  JSON_EXTRACT(@doc, '$.in_stock') AS in_stock;
```

```text
"Widget"  9.99  true
```

Note that string values are returned with surrounding quotes. Use `JSON_UNQUOTE()` or the `->>` operator to strip them.

## Unquoting String Results

```sql
SELECT JSON_UNQUOTE(JSON_EXTRACT(@doc, '$.product')) AS product;
-- Widget  (no quotes)

-- Shorthand using the ->> operator
SELECT @doc->>'$.product' AS product;
-- Widget
```

## Extracting from a Table Column

```sql
CREATE TABLE products (
  id      INT AUTO_INCREMENT PRIMARY KEY,
  details JSON NOT NULL
);

INSERT INTO products (details) VALUES
  ('{"name": "Widget", "price": 9.99,  "tags": ["sale", "new"]}'),
  ('{"name": "Gadget", "price": 24.50, "tags": ["featured"]}'),
  ('{"name": "Doohickey", "price": 4.99, "tags": []}');
```

Extract the name and price from every row:

```sql
SELECT
  id,
  details->>'$.name'       AS name,
  details->'$.price'       AS price,
  details->'$.tags[0]'     AS first_tag
FROM products;
```

## Filtering with JSON_EXTRACT in WHERE

Find products priced above 10:

```sql
SELECT id, details->>'$.name' AS name
FROM products
WHERE JSON_EXTRACT(details, '$.price') > 10;
```

Find products that have a first tag equal to "sale":

```sql
SELECT id, details->>'$.name' AS name
FROM products
WHERE details->>'$.tags[0]' = 'sale';
```

## Extracting Nested Objects

```sql
SET @order = '{
  "id": 101,
  "customer": {"name": "Bob", "email": "bob@example.com"},
  "total": 49.95
}';

SELECT
  JSON_EXTRACT(@order, '$.customer.name')  AS customer_name,
  JSON_EXTRACT(@order, '$.customer.email') AS customer_email,
  JSON_EXTRACT(@order, '$.total')          AS total;
```

## Extracting Multiple Paths at Once

When you provide multiple path arguments, MySQL returns a JSON array of matched values:

```sql
SELECT JSON_EXTRACT(@doc, '$.product', '$.price');
-- ["Widget", 9.99]
```

## NULL Behavior

If the path does not match any key in the document, the result is `NULL`:

```sql
SELECT JSON_EXTRACT('{"a": 1}', '$.b');
-- NULL
```

Use `COALESCE()` to provide a default:

```sql
SELECT COALESCE(details->>'$.sku', 'N/A') AS sku
FROM products;
```

## Generating Indexes on JSON Paths

You cannot index `JSON_EXTRACT()` directly, but you can create a generated column and index that:

```sql
ALTER TABLE products
  ADD COLUMN price DECIMAL(10,2) AS (JSON_EXTRACT(details, '$.price')) VIRTUAL;

CREATE INDEX idx_price ON products (price);
```

This makes range queries on the extracted value use an index.

## Summary

`JSON_EXTRACT(json_doc, path)` retrieves values from JSON documents using JSONPath expressions. Use `JSON_UNQUOTE()` or the `->>` shorthand to get plain strings without JSON encoding. Filter rows with extracted values in `WHERE` clauses, and create generated columns with indexes to speed up repeated JSON path queries.
