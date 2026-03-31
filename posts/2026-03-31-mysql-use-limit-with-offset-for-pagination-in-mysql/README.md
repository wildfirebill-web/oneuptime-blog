# How to Use LIMIT with OFFSET for Pagination in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Pagination, Limit, Offset, Sql

Description: Learn how to implement page-based pagination in MySQL using LIMIT and OFFSET to efficiently retrieve subsets of rows from large datasets.

---

## Introduction

When dealing with large datasets, returning all rows at once is impractical. MySQL provides `LIMIT` and `OFFSET` to fetch a specific number of rows starting at a given position, enabling page-based pagination.

## Basic LIMIT Syntax

```sql
SELECT column1, column2
FROM table_name
LIMIT count;
```

This returns the first `count` rows.

## LIMIT with OFFSET

```sql
SELECT column1, column2
FROM table_name
LIMIT count OFFSET skip;
```

`OFFSET` specifies how many rows to skip before returning results.

## Alternative Shorthand Syntax

MySQL also supports a two-argument form:

```sql
SELECT column1, column2
FROM table_name
LIMIT skip, count;
```

Note the argument order is reversed compared to the `OFFSET` keyword form.

## Implementing Pagination

To build a paginated query, calculate the offset from the page number and page size.

```sql
-- Page 1 (rows 1-10)
SELECT id, name, email
FROM users
ORDER BY id ASC
LIMIT 10 OFFSET 0;

-- Page 2 (rows 11-20)
SELECT id, name, email
FROM users
ORDER BY id ASC
LIMIT 10 OFFSET 10;

-- Page 3 (rows 21-30)
SELECT id, name, email
FROM users
ORDER BY id ASC
LIMIT 10 OFFSET 20;
```

The formula is: `OFFSET = (page_number - 1) * page_size`

## Example with Application Code

In a web application, you typically receive the page number from the client and compute the offset:

```python
page = 3
page_size = 10
offset = (page - 1) * page_size

query = f"SELECT id, name FROM users ORDER BY id ASC LIMIT {page_size} OFFSET {offset}"
```

Or using a parameterized query:

```python
cursor.execute(
    "SELECT id, name FROM users ORDER BY id ASC LIMIT %s OFFSET %s",
    (page_size, offset)
)
```

## Getting Total Count for Pagination UI

To display "Page 3 of 47", you also need the total row count:

```sql
SELECT COUNT(*) AS total FROM users;
```

Then divide by your page size to compute the total pages.

## Using ORDER BY with LIMIT

Always pair `LIMIT` with `ORDER BY` to ensure consistent, predictable results. Without ORDER BY, the returned rows may differ between queries.

```sql
SELECT id, title, created_at
FROM articles
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

## Performance Issues with Large Offsets

The main drawback of OFFSET pagination is performance degradation at large page numbers. MySQL still scans and discards all rows before the offset.

```sql
-- This scans 100,000 rows to return 10
SELECT id, name FROM users ORDER BY id ASC LIMIT 10 OFFSET 100000;
```

Use `EXPLAIN` to see the full scan:

```sql
EXPLAIN SELECT id, name FROM users ORDER BY id ASC LIMIT 10 OFFSET 100000;
```

## Keyset Pagination as an Alternative

For large datasets, keyset (cursor-based) pagination is more efficient:

```sql
-- After fetching the last row with id = 100000
SELECT id, name
FROM users
WHERE id > 100000
ORDER BY id ASC
LIMIT 10;
```

This approach uses an index seek instead of a full offset scan.

## LIMIT in DELETE and UPDATE

`LIMIT` also works with `DELETE` and `UPDATE` to restrict how many rows are affected:

```sql
DELETE FROM logs
WHERE created_at < '2023-01-01'
ORDER BY created_at ASC
LIMIT 1000;
```

## Summary

`LIMIT` with `OFFSET` is the standard way to implement page-based pagination in MySQL. Always use `ORDER BY` alongside it to guarantee consistent results. For very large datasets, consider keyset pagination to avoid performance degradation at high offsets.
