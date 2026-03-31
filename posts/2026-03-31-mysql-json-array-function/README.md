# How to Use JSON_ARRAY() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, Function

Description: Learn how to use MySQL's JSON_ARRAY() function to create JSON arrays from SQL values, with practical examples for building dynamic JSON documents.

---

## What Is JSON_ARRAY()?

MySQL's `JSON_ARRAY()` function constructs a JSON array from a list of values provided as arguments. Each argument becomes an element of the resulting array, preserving the order in which they are passed. This function was introduced in MySQL 5.7.8 alongside native JSON support.

The syntax is straightforward:

```sql
JSON_ARRAY(val1, val2, ..., valN)
```

If called with no arguments, it returns an empty JSON array `[]`.

## Basic Usage

Creating a simple JSON array from literals:

```sql
SELECT JSON_ARRAY(1, 2, 3);
-- Result: [1, 2, 3]

SELECT JSON_ARRAY('apple', 'banana', 'cherry');
-- Result: ["apple", "banana", "cherry"]

SELECT JSON_ARRAY(1, 'two', NULL, TRUE, 3.14);
-- Result: [1, "two", null, true, 3.14]
```

MySQL correctly maps SQL types to their JSON equivalents - integers stay as numbers, strings become JSON strings, and SQL `NULL` becomes JSON `null`.

## Using JSON_ARRAY() with Column Data

You can pass column values as arguments to build arrays dynamically from table rows:

```sql
SELECT
  id,
  JSON_ARRAY(first_name, last_name, email) AS contact_info
FROM users
WHERE active = 1;
```

This is useful when you need to serialize multiple columns into a single JSON field for API responses or data export.

## Nesting Arrays and Objects

`JSON_ARRAY()` can be nested with `JSON_OBJECT()` to build complex documents:

```sql
SELECT JSON_ARRAY(
  JSON_OBJECT('name', 'Alice', 'age', 30),
  JSON_OBJECT('name', 'Bob', 'age', 25)
) AS team_members;
```

Result:

```json
[{"age": 30, "name": "Alice"}, {"age": 25, "name": "Bob"}]
```

## Storing JSON Arrays in a Column

```sql
CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  tags JSON
);

INSERT INTO products (name, tags)
VALUES ('Laptop', JSON_ARRAY('electronics', 'portable', 'work'));
```

Retrieving and filtering:

```sql
SELECT name FROM products
WHERE JSON_CONTAINS(tags, '"electronics"');
```

## Building JSON Arrays from Aggregates

While `JSON_ARRAY()` builds an array from explicit values, combine it with `JSON_ARRAYAGG()` for aggregate use. However, `JSON_ARRAY()` is useful in subqueries:

```sql
SELECT
  order_id,
  JSON_ARRAY(
    (SELECT product_id FROM order_items WHERE order_id = o.order_id LIMIT 1),
    (SELECT quantity FROM order_items WHERE order_id = o.order_id LIMIT 1)
  ) AS first_item_info
FROM orders o;
```

## Type Handling Details

MySQL converts SQL values to JSON types as follows:

```sql
SELECT JSON_ARRAY(
  CAST(1 AS UNSIGNED),   -- JSON number
  CAST('text' AS CHAR),  -- JSON string
  CAST(NULL AS CHAR),    -- JSON null
  NOW()                  -- JSON string (datetime as string)
);
```

Understanding these conversions prevents unexpected results when mixing types.

## Summary

`JSON_ARRAY()` is a foundational function for generating JSON arrays in MySQL. It accepts any number of arguments, handles mixed types, and supports nesting with other JSON functions. Use it to serialize column data, build structured documents, or construct JSON payloads inline in SQL queries.
