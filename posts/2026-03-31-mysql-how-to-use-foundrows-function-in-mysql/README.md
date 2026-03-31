# How to Use FOUND_ROWS() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, FOUND_ROWS, Pagination, SQL Functions

Description: Learn how to use FOUND_ROWS() with SQL_CALC_FOUND_ROWS in MySQL to get the total row count for paginated queries without running a separate COUNT query.

---

## What is FOUND_ROWS()

MySQL's `FOUND_ROWS()` function returns the total number of rows that a `SELECT` statement would have returned if it had no `LIMIT` clause, provided the query used the `SQL_CALC_FOUND_ROWS` modifier.

This is particularly useful for pagination - you can get both the current page of data and the total row count in a single query execution.

Syntax:

```sql
SELECT SQL_CALC_FOUND_ROWS ...columns... FROM table LIMIT offset, count;
SELECT FOUND_ROWS();
```

## Basic Usage

```sql
-- Step 1: Run the paginated query with SQL_CALC_FOUND_ROWS
SELECT SQL_CALC_FOUND_ROWS id, name, salary
FROM employees
WHERE department = 'Engineering'
LIMIT 10 OFFSET 0;

-- Step 2: Get the total count
SELECT FOUND_ROWS() AS total_rows;
-- Returns total Engineering employees, ignoring the LIMIT
```

## Pagination Example

```sql
-- Page 1 (rows 1-10)
SELECT SQL_CALC_FOUND_ROWS id, name, email
FROM users
WHERE active = 1
ORDER BY created_at DESC
LIMIT 10 OFFSET 0;

SELECT FOUND_ROWS() AS total_users;  -- e.g., 250

-- Page 2 (rows 11-20)
SELECT SQL_CALC_FOUND_ROWS id, name, email
FROM users
WHERE active = 1
ORDER BY created_at DESC
LIMIT 10 OFFSET 10;

SELECT FOUND_ROWS() AS total_users;  -- 250 (same total)
```

## FOUND_ROWS() vs Separate COUNT Query

Without `SQL_CALC_FOUND_ROWS`, you need two queries:

```sql
-- Approach 1: Two separate queries (traditional)
SELECT COUNT(*) FROM users WHERE active = 1;
SELECT id, name FROM users WHERE active = 1 LIMIT 10 OFFSET 0;

-- Approach 2: One query + FOUND_ROWS()
SELECT SQL_CALC_FOUND_ROWS id, name FROM users WHERE active = 1 LIMIT 10 OFFSET 0;
SELECT FOUND_ROWS();
```

## Deprecation Notice

`SQL_CALC_FOUND_ROWS` and `FOUND_ROWS()` are deprecated in MySQL 8.0.17+. The recommended modern approach is to run a separate `COUNT(*)` query.

```sql
-- Modern approach (MySQL 8.0+)
SELECT COUNT(*) FROM users WHERE active = 1;
SELECT id, name FROM users WHERE active = 1 LIMIT 10 OFFSET 0;
```

For many workloads, using a window function is another alternative:

```sql
-- Window function approach (MySQL 8.0+)
SELECT
  id,
  name,
  COUNT(*) OVER() AS total_rows
FROM users
WHERE active = 1
ORDER BY created_at DESC
LIMIT 10 OFFSET 0;
```

## Using FOUND_ROWS() in Application Code

In Python:

```python
cursor.execute("""
    SELECT SQL_CALC_FOUND_ROWS id, name
    FROM users
    WHERE active = 1
    LIMIT %s OFFSET %s
""", (page_size, offset))

users = cursor.fetchall()

cursor.execute("SELECT FOUND_ROWS()")
total = cursor.fetchone()[0]
total_pages = (total + page_size - 1) // page_size
```

In Node.js:

```javascript
const [rows] = await conn.execute(
    'SELECT SQL_CALC_FOUND_ROWS id, name FROM users WHERE active = ? LIMIT ? OFFSET ?',
    [1, pageSize, offset]
);

const [[{ total }]] = await conn.execute('SELECT FOUND_ROWS() AS total');
const totalPages = Math.ceil(total / pageSize);
```

## FOUND_ROWS() vs ROW_COUNT()

```sql
-- FOUND_ROWS(): total matching rows in last SELECT (ignoring LIMIT)
SELECT SQL_CALC_FOUND_ROWS * FROM products LIMIT 5;
SELECT FOUND_ROWS();  -- e.g., 500 (total products matching)

-- ROW_COUNT(): rows affected by last DML
UPDATE products SET stock = 0 WHERE stock < 0;
SELECT ROW_COUNT();  -- e.g., 3 (rows actually changed)
```

## Summary

`FOUND_ROWS()` used with `SQL_CALC_FOUND_ROWS` returns the full unfiltered row count for paginated queries in a single scan. While convenient, this feature is deprecated in MySQL 8.0.17+. For new code, prefer a separate `COUNT(*)` query or use `COUNT(*) OVER()` as a window function. `FOUND_ROWS()` must be called immediately after the `SELECT` - any other query in between will reset the value.
