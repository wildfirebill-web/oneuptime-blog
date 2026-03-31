# How to Use JSON_TABLE() in MySQL 8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, JSON_TABLE, SQL, Data Transformation

Description: Learn how to use the JSON_TABLE() function in MySQL 8 to transform JSON arrays and objects into relational rows for standard SQL querying.

---

## What Is JSON_TABLE()

`JSON_TABLE()` is a table-valued function introduced in MySQL 8.0 that extracts data from a JSON document and presents it as a virtual relational table. This allows you to use JSON arrays and objects directly in SQL queries with `JOIN`, `WHERE`, and aggregation.

## Basic Syntax

```sql
JSON_TABLE(
  json_doc,
  path COLUMNS (
    column_name type PATH json_path [options],
    ...
  )
) AS alias
```

## Simple Example - Extracting Array Elements

```sql
SELECT *
FROM JSON_TABLE(
  '[{"name": "Alice", "age": 30}, {"name": "Bob", "age": 25}]',
  '$[*]' COLUMNS (
    name VARCHAR(100) PATH '$.name',
    age  INT          PATH '$.age'
  )
) AS jt;
```

Output:

```text
+-------+-----+
| name  | age |
+-------+-----+
| Alice |  30 |
| Bob   |  25 |
+-------+-----+
```

## Querying JSON Stored in a Table Column

```sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer VARCHAR(100),
  items JSON
);

INSERT INTO orders VALUES
  (1, 'Alice', '[{"product":"Widget","qty":2,"price":9.99},{"product":"Gadget","qty":1,"price":19.99}]'),
  (2, 'Bob',   '[{"product":"Doohickey","qty":5,"price":4.99}]');
```

```sql
SELECT
  o.id,
  o.customer,
  jt.product,
  jt.qty,
  jt.price,
  jt.qty * jt.price AS line_total
FROM orders o,
JSON_TABLE(
  o.items,
  '$[*]' COLUMNS (
    product VARCHAR(100) PATH '$.product',
    qty     INT          PATH '$.qty',
    price   DECIMAL(10,2) PATH '$.price'
  )
) AS jt;
```

Output:

```text
+----+----------+-----------+-----+-------+------------+
| id | customer | product   | qty | price | line_total |
+----+----------+-----------+-----+-------+------------+
|  1 | Alice    | Widget    |   2 |  9.99 |      19.98 |
|  1 | Alice    | Gadget    |   1 | 19.99 |      19.99 |
|  2 | Bob      | Doohickey |   5 |  4.99 |      24.95 |
+----+----------+-----------+-----+-------+------------+
```

## Handling Missing Fields with DEFAULT and ERROR Options

```sql
SELECT *
FROM JSON_TABLE(
  '[{"name": "Alice"}, {"age": 25}]',
  '$[*]' COLUMNS (
    name VARCHAR(100) PATH '$.name' DEFAULT 'Unknown' ON EMPTY,
    age  INT          PATH '$.age'  DEFAULT 0 ON EMPTY NULL ON ERROR
  )
) AS jt;
```

- `DEFAULT 'Unknown' ON EMPTY` - used when the path does not exist in the document.
- `NULL ON ERROR` - used when a type conversion fails.

## Nested Arrays with NESTED PATH

```sql
SELECT *
FROM JSON_TABLE(
  '[{"order_id": 1, "tags": ["urgent", "express"]}, {"order_id": 2, "tags": ["standard"]}]',
  '$[*]' COLUMNS (
    order_id INT PATH '$.order_id',
    NESTED PATH '$.tags[*]' COLUMNS (
      tag VARCHAR(50) PATH '$'
    )
  )
) AS jt;
```

Output:

```text
+----------+----------+
| order_id | tag      |
+----------+----------+
|        1 | urgent   |
|        1 | express  |
|        2 | standard |
+----------+----------+
```

## Using FOR ORDINALITY to Add Row Numbers

```sql
SELECT *
FROM JSON_TABLE(
  '["apple", "banana", "cherry"]',
  '$[*]' COLUMNS (
    row_num FOR ORDINALITY,
    fruit VARCHAR(50) PATH '$'
  )
) AS jt;
```

```text
+---------+--------+
| row_num | fruit  |
+---------+--------+
|       1 | apple  |
|       2 | banana |
|       3 | cherry |
+---------+--------+
```

## Aggregating Extracted JSON Data

```sql
SELECT
  o.customer,
  SUM(jt.qty * jt.price) AS total_order_value
FROM orders o,
JSON_TABLE(
  o.items,
  '$[*]' COLUMNS (
    qty   INT           PATH '$.qty',
    price DECIMAL(10,2) PATH '$.price'
  )
) AS jt
GROUP BY o.customer;
```

## Summary

`JSON_TABLE()` bridges the gap between JSON documents and relational SQL in MySQL 8. It can expand JSON arrays into rows, handle nested arrays with `NESTED PATH`, assign ordinal positions, and gracefully handle missing or invalid values. This makes it ideal for querying semi-structured event data, API responses, and flexible attribute stores stored as JSON columns.
