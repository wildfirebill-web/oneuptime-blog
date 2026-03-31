# How to Implement Database Pagination in REST APIs with MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Pagination, REST API, Cursor, Performance

Description: Implement efficient database pagination in MySQL REST APIs using both offset-based and cursor-based approaches with performance tradeoffs explained.

---

## Pagination Fundamentals

Pagination is essential for REST APIs that return large datasets. Without it, a single request could return millions of rows, overwhelming clients and exhausting server memory. MySQL offers two main pagination strategies: offset-based (using `LIMIT` and `OFFSET`) and cursor-based (using a keyset or cursor).

## Offset-Based Pagination

Offset pagination is simple to implement and supports jumping to arbitrary pages:

```sql
-- Page 3, 20 items per page
SELECT id, user_id, total, status, created_at
FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 40;
```

The REST API query parameters map to SQL:

```javascript
// Node.js Express example
router.get('/orders', async (req, res) => {
  const page  = Math.max(1, parseInt(req.query.page) || 1);
  const limit = Math.min(100, parseInt(req.query.limit) || 20);
  const offset = (page - 1) * limit;

  const [rows] = await pool.query(
    'SELECT id, user_id, total, status, created_at FROM orders ORDER BY created_at DESC, id DESC LIMIT ? OFFSET ?',
    [limit, offset]
  );

  const [[{ total }]] = await pool.query('SELECT COUNT(*) AS total FROM orders');

  res.json({
    data:       rows,
    pagination: {
      page,
      limit,
      total:       total,
      total_pages: Math.ceil(total / limit),
      has_next:    page * limit < total,
      has_prev:    page > 1,
    },
  });
});
```

### Offset Pagination Response

```json
{
  "data": [...],
  "pagination": {
    "page": 3,
    "limit": 20,
    "total": 1500,
    "total_pages": 75,
    "has_next": true,
    "has_prev": true
  }
}
```

## Performance Problem with Large Offsets

Offset pagination becomes slow at large offsets because MySQL must scan and discard all rows up to the offset:

```sql
-- EXPLAIN shows 10000 rows examined for OFFSET 10000
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
```

## Cursor-Based Pagination

Cursor pagination avoids this by using the last row's values as a starting point:

```sql
-- First page (no cursor)
SELECT id, user_id, total, created_at
FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page (cursor = last row's created_at and id from previous response)
SELECT id, user_id, total, created_at
FROM orders
WHERE (created_at, id) < ('2026-03-15 10:00:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

```javascript
// Cursor pagination implementation
router.get('/orders', async (req, res) => {
  const limit  = Math.min(100, parseInt(req.query.limit) || 20);
  const cursor = req.query.cursor;

  let query;
  let params;

  if (cursor) {
    const [cursorCreatedAt, cursorId] = Buffer.from(cursor, 'base64')
      .toString('utf-8').split('|');
    query = `SELECT id, user_id, total, created_at FROM orders
             WHERE (created_at < ? OR (created_at = ? AND id < ?))
             ORDER BY created_at DESC, id DESC
             LIMIT ?`;
    params = [cursorCreatedAt, cursorCreatedAt, parseInt(cursorId), limit + 1];
  } else {
    query  = 'SELECT id, user_id, total, created_at FROM orders ORDER BY created_at DESC, id DESC LIMIT ?';
    params = [limit + 1];
  }

  const [rows] = await pool.query(query, params);
  const hasNext = rows.length > limit;
  const data = hasNext ? rows.slice(0, limit) : rows;

  const lastRow = data[data.length - 1];
  const nextCursor = hasNext
    ? Buffer.from(`${lastRow.created_at.toISOString()}|${lastRow.id}`).toString('base64')
    : null;

  res.json({ data, pagination: { next_cursor: nextCursor, has_next: hasNext } });
});
```

## Index Requirements

Both strategies require the correct index for efficiency:

```sql
-- Index for cursor pagination on (created_at, id)
ALTER TABLE orders ADD INDEX idx_created_at_id (created_at DESC, id DESC);
```

## Summary

Offset pagination is simple and supports random page access but degrades at large offsets. Cursor-based pagination maintains consistent O(log n) performance regardless of position by using a keyset comparison instead of scanning discarded rows. For APIs with large datasets or infinite scroll requirements, cursor pagination is the better choice.
