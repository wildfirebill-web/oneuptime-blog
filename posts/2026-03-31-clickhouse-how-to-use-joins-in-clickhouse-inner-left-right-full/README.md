# How to Use JOINs in ClickHouse (INNER, LEFT, RIGHT, FULL)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, JOINs, Query Optimization, Analytics

Description: Learn how to use INNER, LEFT, RIGHT, and FULL JOINs in ClickHouse, understand the performance implications, and apply best practices for efficient joins.

---

## JOIN Types in ClickHouse

ClickHouse supports all standard SQL JOIN types, with some important performance considerations due to its distributed columnar architecture.

| JOIN Type | Returns |
|-----------|---------|
| `INNER JOIN` | Matching rows from both tables |
| `LEFT JOIN` | All left rows + matching right rows (NULL for no match) |
| `RIGHT JOIN` | All right rows + matching left rows (NULL for no match) |
| `FULL OUTER JOIN` | All rows from both tables (NULLs for non-matches) |
| `CROSS JOIN` | Cartesian product |
| `SEMI JOIN` | Left rows that have at least one match in right |
| `ANTI JOIN` | Left rows that have NO match in right |

## INNER JOIN

```sql
-- Basic INNER JOIN: only matching rows
SELECT
    o.order_id,
    o.amount,
    u.name AS customer_name,
    u.email
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id
WHERE o.order_date >= today() - 30
ORDER BY o.order_date DESC;

-- INNER JOIN with multiple conditions
SELECT a.*, b.category
FROM products a
INNER JOIN categories b ON a.category_id = b.id AND b.active = 1;
```

## LEFT JOIN

```sql
-- LEFT JOIN: all orders, with user info where available
SELECT
    o.order_id,
    o.amount,
    u.name AS customer_name,
    -- NULL if no matching user
    ifNull(u.email, 'unknown@example.com') AS email
FROM orders o
LEFT JOIN users u ON o.user_id = u.user_id;

-- LEFT JOIN to find orders with no users (orphaned records)
SELECT o.order_id, o.user_id
FROM orders o
LEFT JOIN users u ON o.user_id = u.user_id
WHERE u.user_id IS NULL;  -- no matching user
```

## RIGHT JOIN

```sql
-- RIGHT JOIN: all users, with orders where available
SELECT
    u.user_id,
    u.name,
    count(o.order_id) AS order_count,
    ifNull(sum(o.amount), 0) AS total_spent
FROM orders o
RIGHT JOIN users u ON o.user_id = u.user_id
GROUP BY u.user_id, u.name
ORDER BY total_spent DESC;
```

## FULL OUTER JOIN

```sql
-- FULL OUTER JOIN: all rows from both tables
SELECT
    COALESCE(a.date, b.date) AS date,
    ifNull(a.web_sessions, 0) AS web_sessions,
    ifNull(b.mobile_sessions, 0) AS mobile_sessions
FROM web_analytics a
FULL OUTER JOIN mobile_analytics b ON a.date = b.date
ORDER BY date;
```

## CROSS JOIN

```sql
-- CROSS JOIN: cartesian product (use carefully)
-- Good for generating combinations
SELECT
    a.size,
    b.color,
    concat(a.size, '-', b.color) AS variant
FROM sizes a
CROSS JOIN colors b
ORDER BY variant;

-- With implicit cross join syntax (comma)
SELECT a.size, b.color
FROM sizes a, colors b
LIMIT 100;
```

## SEMI JOIN and ANTI JOIN

```sql
-- LEFT SEMI JOIN: left rows where a match exists in right
SELECT u.user_id, u.name
FROM users u
LEFT SEMI JOIN orders o ON u.user_id = o.user_id;
-- Returns users who have at least one order

-- LEFT ANTI JOIN: left rows where NO match exists
SELECT u.user_id, u.name
FROM users u
LEFT ANTI JOIN orders o ON u.user_id = o.user_id;
-- Returns users with no orders

-- Equivalent to SEMI with IN/EXISTS
SELECT user_id, name
FROM users
WHERE user_id IN (SELECT DISTINCT user_id FROM orders);
```

## JOIN Performance Best Practices

### 1. Put the larger table on the left

```sql
-- ClickHouse loads the RIGHT table into memory (by default)
-- Put LARGER table on LEFT, SMALLER table on RIGHT
SELECT big.*, small.name
FROM large_events_table big
LEFT JOIN small_lookup small ON big.key = small.key;
```

### 2. Use JOIN with dictionary for small lookups

```sql
-- Instead of JOIN for small lookup tables:
SELECT
    event_id,
    dictGet('country_dict', 'name', country_code) AS country_name
FROM events;

-- This is faster than:
SELECT e.event_id, c.name
FROM events e
JOIN countries c ON e.country_code = c.code;
```

### 3. Filter before joining

```sql
-- Filter early to reduce join size
SELECT o.order_id, u.email
FROM (
    SELECT * FROM orders WHERE order_date >= today() - 7
) o
INNER JOIN users u ON o.user_id = u.user_id;
```

### 4. Use join_algorithm setting

```sql
-- Hash join (default): loads right table into hash table
SET join_algorithm = 'hash';

-- Partial merge join: for large tables that don't fit in memory
SET join_algorithm = 'partial_merge';

-- Grace hash join: spills to disk when memory exceeded
SET join_algorithm = 'grace_hash';
SET grace_hash_join_initial_buckets = 16;
```

## Distributed JOINs

```sql
-- For distributed tables, use GLOBAL JOIN to broadcast right table
SELECT a.order_id, b.user_name
FROM distributed_orders a
GLOBAL INNER JOIN distributed_users b ON a.user_id = b.user_id;

-- GLOBAL prevents each shard from querying every other shard
-- Right table is collected to coordinator then broadcast
```

## Practical Example: E-commerce Funnel Analysis

```sql
WITH
    checkouts AS (
        SELECT DISTINCT user_id FROM events WHERE event = 'checkout'
    ),
    purchases AS (
        SELECT DISTINCT user_id FROM events WHERE event = 'purchase'
    )
SELECT
    c.user_id,
    (p.user_id IS NOT NULL) AS completed_purchase
FROM checkouts c
LEFT JOIN purchases p ON c.user_id = p.user_id;

-- Funnel summary
SELECT
    count() AS total_checkouts,
    countIf(p.user_id IS NOT NULL) AS completed_purchases,
    round(countIf(p.user_id IS NOT NULL) / count() * 100, 1) AS conversion_rate
FROM checkouts c
LEFT JOIN purchases p ON c.user_id = p.user_id;
```

## Summary

ClickHouse supports all standard SQL JOIN types - INNER, LEFT, RIGHT, FULL OUTER, CROSS, SEMI, and ANTI. For performance, keep the right-side table smaller since it is loaded into memory, use dictionaries instead of JOINs for small lookup tables, filter rows before joining, and choose the appropriate `join_algorithm` setting for your data size. For distributed setups, use `GLOBAL JOIN` to avoid cross-shard queries.
