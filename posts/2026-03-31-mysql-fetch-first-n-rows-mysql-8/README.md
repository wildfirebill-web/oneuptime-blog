# How to Use FETCH FIRST N ROWS in MySQL 8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Pagination, Standard, Query

Description: Learn how to use the SQL standard FETCH FIRST N ROWS syntax in MySQL 8.0 for cleaner, portable result set limiting and pagination.

---

## SQL Standard Pagination Syntax

MySQL has long supported `LIMIT` and `OFFSET` for pagination, but these are MySQL extensions to SQL. MySQL 8.0 added support for the SQL standard `FETCH FIRST` / `FETCH NEXT` syntax, making MySQL queries more portable across database systems.

## Basic FETCH FIRST Syntax

```sql
-- SQL standard syntax - fetch first 10 rows
SELECT id, name, price
FROM products
ORDER BY price DESC
FETCH FIRST 10 ROWS ONLY;

-- Equivalent MySQL-specific syntax
SELECT id, name, price
FROM products
ORDER BY price DESC
LIMIT 10;
```

Both are valid in MySQL 8.0. The `FETCH FIRST` version is the SQL:2008 standard.

## FETCH NEXT for Pagination

Use `OFFSET` combined with `FETCH NEXT` for pagination:

```sql
-- Page 1: rows 1-10
SELECT id, name, price
FROM products
ORDER BY price DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;

-- Page 2: rows 11-20
SELECT id, name, price
FROM products
ORDER BY price DESC
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;

-- Page 3: rows 21-30
SELECT id, name, price
FROM products
ORDER BY price DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

## Syntax Variations

MySQL 8.0 supports both `ROW` and `ROWS` as well as `FIRST` and `NEXT` interchangeably:

```sql
-- All of these are equivalent
SELECT * FROM products ORDER BY id FETCH FIRST 5 ROWS ONLY;
SELECT * FROM products ORDER BY id FETCH FIRST 5 ROW ONLY;
SELECT * FROM products ORDER BY id FETCH NEXT 5 ROWS ONLY;
SELECT * FROM products ORDER BY id FETCH NEXT 5 ROW ONLY;

-- OFFSET also accepts ROW or ROWS
SELECT * FROM products ORDER BY id
OFFSET 10 ROW FETCH NEXT 5 ROWS ONLY;
```

## Using WITH TIES

MySQL 8.0.4+ supports `WITH TIES`, which returns extra rows that tie with the last row in the result set:

```sql
-- Return top 3 products by price, including all ties for 3rd place
SELECT id, name, price
FROM products
ORDER BY price DESC
FETCH FIRST 3 ROWS WITH TIES;
```

If three products share the 3rd highest price, all three are returned even though you asked for 3 rows.

## Practical Pagination Example

```sql
-- Create a reusable pagination query
-- Page number and page size as variables
SET @page = 2;
SET @page_size = 10;
SET @offset = (@page - 1) * @page_size;

SELECT id, name, price, created_at
FROM products
WHERE category = 'electronics'
ORDER BY created_at DESC
LIMIT @page_size OFFSET @offset;

-- Or using FETCH FIRST syntax with a prepared statement
SET @offset_val = 10;
SET @fetch_count = 10;

PREPARE stmt FROM
    'SELECT id, name, price FROM products
     ORDER BY price DESC
     OFFSET ? ROWS FETCH NEXT ? ROWS ONLY';

EXECUTE stmt USING @offset_val, @fetch_count;
DEALLOCATE PREPARE stmt;
```

## Checking Compatibility

```sql
-- FETCH FIRST requires MySQL 8.0.4 or later
SELECT VERSION();
```

## Performance Considerations

Both `LIMIT/OFFSET` and `FETCH FIRST` have the same performance characteristics - MySQL must scan and discard offset rows before returning results. For deep pagination on large tables, consider keyset pagination instead:

```sql
-- Efficient keyset pagination (no offset scanning)
SELECT id, name, price
FROM products
WHERE id > :last_seen_id
ORDER BY id
FETCH FIRST 10 ROWS ONLY;
```

## Summary

The `FETCH FIRST N ROWS ONLY` syntax in MySQL 8.0 brings SQL standard compliance for result set limiting and pagination. It is functionally equivalent to `LIMIT/OFFSET` but more portable across database systems. The addition of `WITH TIES` is particularly useful for top-N queries where ties in the boundary condition should all be included in results.
