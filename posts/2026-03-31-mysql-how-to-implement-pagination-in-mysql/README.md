# How to Implement Pagination in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Pagination, Performance, SQL, API

Description: Learn how to implement efficient pagination in MySQL using LIMIT/OFFSET and keyset pagination, and understand why OFFSET pagination degrades at scale.

---

## Why Pagination Matters

Returning all rows from a large table in a single query is impractical. Pagination splits results into pages, reducing response time and memory usage. However, not all pagination techniques perform equally well at scale.

## Method 1: LIMIT/OFFSET Pagination

The most common approach:

```sql
-- Page 1 (items 1-10)
SELECT id, name, created_at
FROM orders
ORDER BY id
LIMIT 10 OFFSET 0;

-- Page 2 (items 11-20)
SELECT id, name, created_at
FROM orders
ORDER BY id
LIMIT 10 OFFSET 10;

-- Page N
SELECT id, name, created_at
FROM orders
ORDER BY id
LIMIT 10 OFFSET (N-1) * 10;
```

In a web application, the offset is typically computed from a page number parameter:

```python
def get_page(page: int, per_page: int = 10):
    offset = (page - 1) * per_page
    cursor.execute("""
        SELECT id, name, created_at
        FROM orders
        ORDER BY id
        LIMIT %s OFFSET %s
    """, (per_page, offset))
    return cursor.fetchall()
```

### The OFFSET Problem

LIMIT/OFFSET has a severe performance problem at large offsets. For `LIMIT 10 OFFSET 100000`, MySQL scans 100,010 rows and discards 100,000 of them:

```sql
-- Slow at large offsets: MySQL scans all 100,010 rows
EXPLAIN SELECT id, name FROM orders ORDER BY id LIMIT 10 OFFSET 100000;
```

```text
type: index
rows: 100010   -- MySQL examines 100,010 rows!
```

## Method 2: Keyset Pagination (Cursor-Based)

Keyset pagination uses the last-seen primary key as a cursor instead of an offset. This is O(1) regardless of how deep into the result set you are:

```sql
-- First page
SELECT id, name, created_at
FROM orders
ORDER BY id ASC
LIMIT 10;

-- Next page: use the last id from the previous page (e.g., last_id = 10)
SELECT id, name, created_at
FROM orders
WHERE id > 10
ORDER BY id ASC
LIMIT 10;

-- Continue with the new last_id
SELECT id, name, created_at
FROM orders
WHERE id > 20
ORDER BY id ASC
LIMIT 10;
```

### EXPLAIN Comparison

```sql
-- Keyset pagination (fast - uses index)
EXPLAIN SELECT id, name FROM orders WHERE id > 100000 ORDER BY id LIMIT 10;
```

```text
type: range
key: PRIMARY
rows: 10    -- Only examines 10 rows!
```

This makes keyset pagination dramatically faster than OFFSET for large datasets.

## Method 3: Deferred Join (Optimized OFFSET)

When you need page numbers (not just next/prev), combine a covering index scan with a join:

```sql
SELECT o.*
FROM orders o
INNER JOIN (
  SELECT id
  FROM orders
  ORDER BY id
  LIMIT 10 OFFSET 100000
) ids USING (id);
```

The inner query uses a covering index on `id` only (fast), and the outer query fetches full rows only for the 10 matching ids.

## Getting Total Count for Pagination UI

```sql
-- Total rows for page count display
SELECT COUNT(*) AS total FROM orders WHERE status = 'active';

-- With estimated count for large tables (approximate but fast)
SELECT table_rows AS approx_total
FROM information_schema.tables
WHERE table_schema = 'mydb' AND table_name = 'orders';
```

## Full Pagination Response (REST API Pattern)

```python
def get_orders_page(last_id: int = 0, per_page: int = 10):
    cursor.execute("""
        SELECT id, customer_id, total, created_at
        FROM orders
        WHERE id > %s
        ORDER BY id ASC
        LIMIT %s
    """, (last_id, per_page + 1))  # Fetch one extra to detect next page

    rows = cursor.fetchall()
    has_next = len(rows) > per_page
    items = rows[:per_page]

    return {
        "items": items,
        "has_next": has_next,
        "next_cursor": items[-1]["id"] if items else None
    }
```

## Pagination with Multi-Column Sort

For pagination with non-unique sort columns (e.g., `created_at`), include the primary key as a tiebreaker:

```sql
-- Get orders sorted by created_at, using (created_at, id) as cursor
SELECT id, created_at, total
FROM orders
WHERE (created_at, id) > ('2026-01-01 10:00:00', 12345)
ORDER BY created_at ASC, id ASC
LIMIT 10;
```

## Summary

Use LIMIT/OFFSET pagination for small datasets or when random page access is required. Switch to keyset pagination for large tables or infinite-scroll UIs where sequential page navigation is acceptable. The keyset approach eliminates the O(N) offset scan penalty and scales to millions of rows with consistent performance. Use a deferred join as a middle ground when you need OFFSET-based navigation but need better performance.
