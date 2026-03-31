# How to Translate MySQL Queries to ClickHouse SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MySQL, SQL Translation, Migration, Analytics

Description: Translate common MySQL SQL patterns to ClickHouse SQL, covering GROUP BY, window functions, subqueries, and aggregate functions with practical examples.

---

Migrating analytics workloads from MySQL to ClickHouse requires adjusting SQL syntax and rethinking query patterns. ClickHouse SQL is ANSI-compatible but has important differences.

## Basic SELECT and Filtering

MySQL and ClickHouse are nearly identical for simple queries. The main difference is ClickHouse's column-oriented engine rewards selecting fewer columns.

```sql
-- MySQL
SELECT user_id, email, created_at FROM users WHERE status = 'active';

-- ClickHouse (identical, but avoid SELECT * for performance)
SELECT user_id, email, created_at FROM users WHERE status = 'active';
```

## GROUP BY Differences

MySQL allows selecting non-aggregated columns not in `GROUP BY`. ClickHouse requires them to be in `GROUP BY` or aggregated:

```sql
-- MySQL (allowed but ambiguous)
SELECT user_id, name, count(*) FROM orders GROUP BY user_id;

-- ClickHouse (explicit)
SELECT user_id, any(name), count() FROM orders GROUP BY user_id;
```

## NULL Handling

ClickHouse functions handle NULLs differently. Use `isNull()` and `ifNull()`:

```sql
-- MySQL
SELECT IFNULL(revenue, 0) FROM sales;

-- ClickHouse
SELECT ifNull(revenue, 0) FROM sales;
```

## Date Functions

```sql
-- MySQL: last 7 days
SELECT * FROM events WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY);

-- ClickHouse
SELECT * FROM events WHERE ts >= now() - INTERVAL 7 DAY;

-- MySQL: extract month
SELECT MONTH(created_at) FROM events;

-- ClickHouse
SELECT toMonth(ts) FROM events;
```

## LIMIT with OFFSET

```sql
-- MySQL
SELECT * FROM events ORDER BY ts DESC LIMIT 10 OFFSET 20;

-- ClickHouse (identical)
SELECT * FROM events ORDER BY ts DESC LIMIT 10 OFFSET 20;
```

## Window Functions

ClickHouse supports window functions similarly to MySQL 8.0:

```sql
-- Both MySQL 8+ and ClickHouse
SELECT
    user_id,
    ts,
    row_number() OVER (PARTITION BY user_id ORDER BY ts) AS rn
FROM events;
```

## Replacing MySQL EXPLAIN

```sql
-- MySQL
EXPLAIN SELECT count(*) FROM events WHERE user_id = 42;

-- ClickHouse
EXPLAIN SELECT count() FROM events WHERE user_id = 42;
-- Or for more detail
EXPLAIN PIPELINE SELECT count() FROM events WHERE user_id = 42;
```

## Key Incompatibilities

| MySQL | ClickHouse Alternative |
|---|---|
| `AUTO_INCREMENT` | No auto-increment; use `generateUUIDv4()` or sequences |
| `UPDATE` / `DELETE` | `ALTER TABLE ... UPDATE/DELETE` (async mutations) |
| Transactions | Limited; avoid multi-row transactions |
| `JOIN` on arbitrary columns | Pre-join using dictionaries for dimension tables |

## Summary

Most MySQL SELECT queries translate directly to ClickHouse with minor syntax adjustments. The biggest changes are around mutations (no inline UPDATE/DELETE), GROUP BY strictness, and replacing MySQL-specific date functions with ClickHouse equivalents.
