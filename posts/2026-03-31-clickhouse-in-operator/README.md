# How to Use the IN Operator Effectively in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, IN Operator, Subquery, SQL, Performance

Description: Learn how to use the IN operator in ClickHouse effectively, including set literals, subqueries, global IN for distributed queries, and performance tips.

---

## Basic IN with a Value List

The `IN` operator checks whether a column value is present in a fixed set. It returns `UInt8` (1 or 0).

```sql
SELECT user_id, event_type, event_time
FROM events
WHERE event_type IN ('purchase', 'refund', 'cart_add')
  AND event_date = today()
```

For large sets, `IN` with a tuple is faster than multiple `OR` conditions because ClickHouse can use a hash set.

## IN with a Subquery

```sql
SELECT *
FROM orders
WHERE user_id IN (
  SELECT user_id FROM users WHERE plan = 'premium'
)
```

ClickHouse executes the subquery once, builds a hash set in memory, and uses it to filter the outer query.

## NOT IN

```sql
SELECT user_id, last_activity
FROM users
WHERE user_id NOT IN (
  SELECT DISTINCT user_id FROM banned_users
)
```

Be careful with `NOT IN` when the subquery may return `NULL` values - any `NULL` in the set causes the entire `NOT IN` result to be `NULL`, effectively hiding rows.

## GLOBAL IN for Distributed Queries

On ClickHouse clusters, a regular `IN` subquery is executed on every shard. `GLOBAL IN` executes the subquery once on the initiating server, then sends the result set to all shards.

```sql
SELECT *
FROM distributed_events
WHERE user_id GLOBAL IN (
  SELECT user_id FROM local_premium_users
)
```

Use `GLOBAL IN` to avoid N subquery executions, one per shard.

## IN with Arrays

You can use `has` or `hasAny` to check if an array column contains a value, which is conceptually similar to `IN` for array data.

```sql
-- Check if a tag appears in the tags array column
SELECT post_id, title
FROM posts
WHERE has(tags, 'clickhouse')

-- Check if any tag from the list is present
SELECT post_id
FROM posts
WHERE hasAny(tags, ['clickhouse', 'analytics', 'olap'])
```

## Performance Tips

- Keep `IN` sets small when possible. Very large in-memory sets slow query planning.
- For very large reference datasets, prefer a ClickHouse Dictionary over a subquery - the lookup is O(1).
- Always place the most selective `IN` condition first in a compound `WHERE`.

```sql
-- Prefer using a dictionary for large lookups
SELECT o.order_id, d.country
FROM orders AS o
JOIN country_dict AS d ON o.country_code = d.code
-- Rather than: WHERE o.country_code IN (SELECT code FROM large_country_table)
```

## Summary

The `IN` operator in ClickHouse accepts value lists, subqueries, and works with `NOT IN` for exclusion. Use `GLOBAL IN` on distributed clusters to prevent redundant subquery execution per shard. For large reference sets, dictionaries outperform `IN` subqueries. Avoid `NOT IN` with subqueries that may return `NULL` values.
