# How to Implement Sorting in REST APIs with MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Sorting, REST API, Index, Query Builder

Description: Implement safe, performant sorting in MySQL REST APIs by validating sort columns against an allowlist and creating supporting indexes.

---

## Sorting in REST APIs

Sorting is a fundamental feature of REST APIs that list resources. A typical request looks like `GET /api/orders?sort=total&dir=desc`. Implementing sorting safely requires validating column names against an allowlist, since column names cannot be parameterized in SQL and must be interpolated directly into the query string.

## Why You Cannot Parameterize Column Names

SQL parameters work for values, not for identifiers (column names, table names). This means you cannot do:

```sql
-- This does NOT work - ? is treated as a string value, not a column name
SELECT * FROM orders ORDER BY ? DESC;
```

Instead, you must validate the column name against an allowlist before interpolating:

```javascript
// Safe sorting implementation in Node.js
const ALLOWED_SORT_COLUMNS = {
  id:         'id',
  total:      'total',
  status:     'status',
  created_at: 'created_at',
  user_id:    'user_id',
};

const ALLOWED_DIRECTIONS = new Set(['asc', 'desc']);

router.get('/orders', async (req, res) => {
  const rawSort = req.query.sort || 'created_at';
  const rawDir  = (req.query.dir || 'desc').toLowerCase();

  // Validate against allowlist
  const sortColumn = ALLOWED_SORT_COLUMNS[rawSort];
  if (!sortColumn) {
    return res.status(400).json({
      error: `Invalid sort column. Allowed: ${Object.keys(ALLOWED_SORT_COLUMNS).join(', ')}`,
    });
  }

  const sortDir = ALLOWED_DIRECTIONS.has(rawDir) ? rawDir.toUpperCase() : 'DESC';

  // Safe to interpolate because we validated against the allowlist
  const [rows] = await pool.query(
    `SELECT id, user_id, total, status, created_at
     FROM orders
     ORDER BY ${sortColumn} ${sortDir}, id DESC
     LIMIT ? OFFSET ?`,
    [20, 0]
  );

  res.json({ data: rows, sort: { column: sortColumn, direction: sortDir } });
});
```

## Multi-Column Sorting

Support sorting by multiple columns for more flexible APIs:

```javascript
// ?sort=status,total&dir=asc,desc
function parseSortParams(sortParam, dirParam, allowedColumns) {
  const columns   = (sortParam || 'created_at').split(',');
  const directions = (dirParam || 'desc').split(',');

  const orderClauses = columns.map((col, i) => {
    const validCol = allowedColumns[col.trim()];
    if (!validCol) return null;
    const dir = (directions[i] || 'desc').toLowerCase() === 'asc' ? 'ASC' : 'DESC';
    return `${validCol} ${dir}`;
  }).filter(Boolean);

  // Always add a tiebreaker
  if (!orderClauses.includes('id ASC') && !orderClauses.includes('id DESC')) {
    orderClauses.push('id DESC');
  }

  return orderClauses.join(', ');
}

const orderBy = parseSortParams(req.query.sort, req.query.dir, ALLOWED_SORT_COLUMNS);
const [rows] = await pool.query(
  `SELECT * FROM orders ORDER BY ${orderBy} LIMIT ? OFFSET ?`,
  [limit, offset]
);
```

## Creating Indexes for Sort Columns

Sorting is only fast if MySQL can use an index. Create composite indexes for common sort patterns:

```sql
-- Index for default sort (created_at DESC, id DESC)
ALTER TABLE orders ADD INDEX idx_created_at_id (created_at DESC, id DESC);

-- Index for sorting by total with status filter
ALTER TABLE orders ADD INDEX idx_status_total (status, total DESC);

-- Index for user-scoped sorts
ALTER TABLE orders ADD INDEX idx_user_id_created_at (user_id, created_at DESC);
```

Verify the index is used:

```sql
EXPLAIN SELECT id, total, status, created_at
FROM orders
WHERE status = 'pending'
ORDER BY total DESC
LIMIT 20;
```

## Communicating Sort Options to API Consumers

Document available sort options in your API response metadata:

```json
{
  "data": [...],
  "meta": {
    "sort": {
      "column": "total",
      "direction": "DESC",
      "available_columns": ["id", "total", "status", "created_at"]
    }
  }
}
```

## Summary

Implementing sorting in MySQL REST APIs safely requires validating sort columns against an allowlist before interpolating them into queries, since column names cannot be parameterized. Multi-column sort support improves flexibility, while composite indexes on common sort column combinations ensure MySQL uses efficient index scans rather than filesorts. Always add a tiebreaker column (typically `id`) to ensure deterministic ordering.
