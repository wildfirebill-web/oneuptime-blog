# How to Use JSON_TABLE() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Table, Json, Json Functions, Sql

Description: Learn how to use MySQL's JSON_TABLE() function to transform JSON arrays and objects into relational rows and columns for use in standard SQL queries.

---

## Overview

`JSON_TABLE()` is a powerful MySQL 8.0 function that transforms a JSON document into a relational table, allowing you to use standard SQL operations like `JOIN`, `WHERE`, `GROUP BY`, and `ORDER BY` on JSON data. It bridges the gap between document-oriented and relational data.

## Basic Syntax

```sql
JSON_TABLE(
  expr,
  path
  COLUMNS (
    column_name column_type PATH col_path [DEFAULT default ON EMPTY | ON ERROR],
    ...
  )
) [AS] alias
```

## Basic Examples

```sql
-- Expand a JSON array into rows
SELECT jt.*
FROM JSON_TABLE(
  '[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]',
  '$[*]'
  COLUMNS (
    id   INT          PATH '$.id',
    name VARCHAR(100) PATH '$.name'
  )
) jt;
-- Returns:
-- id | name
-- 1  | Alice
-- 2  | Bob

-- Expand from a JSON column in a table
CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  order_data JSON
);

INSERT INTO orders (order_data) VALUES
('{"items":[{"sku":"A1","qty":2},{"sku":"B2","qty":1}],"total":150}'),
('{"items":[{"sku":"C3","qty":5}],"total":75}');

SELECT o.id AS order_id, jt.sku, jt.qty
FROM orders o,
JSON_TABLE(
  o.order_data,
  '$.items[*]'
  COLUMNS (
    sku VARCHAR(20) PATH '$.sku',
    qty INT         PATH '$.qty'
  )
) jt;
```

## COLUMNS Clause Options

```sql
-- PATH: extract a value at a path
-- FOR ORDINALITY: generate a row number
-- DEFAULT ... ON EMPTY: value to use if path has no value
-- DEFAULT ... ON ERROR: value to use if type conversion fails
-- NESTED PATH: recurse into nested arrays

SELECT jt.*
FROM JSON_TABLE(
  '[{"name":"Alice","score":null},{"name":"Bob"}]',
  '$[*]'
  COLUMNS (
    row_num  FOR ORDINALITY,
    name     VARCHAR(100) PATH '$.name',
    score    INT PATH '$.score'
      DEFAULT 0 ON EMPTY
      DEFAULT -1 ON ERROR
  )
) jt;
```

## NESTED PATH for Nested Arrays

```sql
SELECT jt.*
FROM JSON_TABLE(
  '[
    {"dept":"Eng","employees":["Alice","Bob"]},
    {"dept":"HR","employees":["Carol"]}
  ]',
  '$[*]'
  COLUMNS (
    dept VARCHAR(50) PATH '$.dept',
    NESTED PATH '$.employees[*]'
    COLUMNS (
      employee VARCHAR(100) PATH '$'
    )
  )
) jt;
-- Returns:
-- dept | employee
-- Eng  | Alice
-- Eng  | Bob
-- HR   | Carol
```

## Joining JSON_TABLE() with Regular Tables

```sql
CREATE TABLE products (
  sku VARCHAR(20) PRIMARY KEY,
  price DECIMAL(10,2)
);

INSERT INTO products VALUES ('A1', 50.00), ('B2', 100.00), ('C3', 15.00);

-- Join expanded JSON items with the products table
SELECT o.id AS order_id, jt.sku, jt.qty, p.price,
  jt.qty * p.price AS line_total
FROM orders o,
JSON_TABLE(
  o.order_data,
  '$.items[*]'
  COLUMNS (
    sku VARCHAR(20) PATH '$.sku',
    qty INT PATH '$.qty'
  )
) jt
JOIN products p ON p.sku = jt.sku;
```

## Aggregating Expanded JSON Data

```sql
-- Total quantity ordered per SKU across all orders
SELECT jt.sku, SUM(jt.qty) AS total_qty
FROM orders o,
JSON_TABLE(
  o.order_data,
  '$.items[*]'
  COLUMNS (
    sku VARCHAR(20) PATH '$.sku',
    qty INT PATH '$.qty'
  )
) jt
GROUP BY jt.sku
ORDER BY total_qty DESC;
```

## Practical Example: Normalizing a JSON Import

```sql
-- Import JSON data and normalize into a relational table
CREATE TABLE normalized_items AS
SELECT o.id AS order_id, jt.sku, jt.qty
FROM orders o,
JSON_TABLE(
  o.order_data,
  '$.items[*]'
  COLUMNS (
    sku VARCHAR(20) PATH '$.sku',
    qty INT PATH '$.qty'
  )
) jt;
```

## Summary

`JSON_TABLE()` is MySQL 8.0's most powerful JSON function, converting JSON documents into relational rows and columns that can be queried with standard SQL. Use it to expand JSON arrays into rows, join JSON data with regular tables, aggregate over JSON contents, and normalize stored JSON into relational structures. The `NESTED PATH` clause handles arrays within arrays, and `FOR ORDINALITY` generates row numbers.
