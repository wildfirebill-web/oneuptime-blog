# How to Use any_join_distinct_right_table_keys in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Database, SQL, Query Optimization, Configuration

Description: Learn how any_join_distinct_right_table_keys works in ClickHouse to control duplicate handling in ANY JOIN semantics and produce correct results when joining against non-unique keys.

---

ClickHouse supports a non-standard `ANY` JOIN type that returns at most one matching row from the right table for each row in the left table. When the right table contains duplicate keys, the behavior of `ANY JOIN` depends on the `any_join_distinct_right_table_keys` setting. Understanding this setting is important for producing predictable results when your join key is not unique in the right table.

## What ANY JOIN Does

Standard SQL JOIN types (`INNER`, `LEFT`, `RIGHT`, `FULL`) return all matching rows. If the right table has 3 rows with the same key, a left row with that key will produce 3 output rows.

ClickHouse `ANY JOIN` returns at most one right-table row per left-table row. This is useful for dimension table lookups where you want exactly one enriching record per event, not a row explosion:

```sql
-- Standard LEFT JOIN: may produce multiple rows if users.user_id is not unique
SELECT e.event_type, u.user_name
FROM events AS e
LEFT JOIN users AS u ON e.user_id = u.user_id;

-- ANY LEFT JOIN: returns exactly one match per event row
SELECT e.event_type, u.user_name
FROM events AS e
ANY LEFT JOIN users AS u ON e.user_id = u.user_id;
```

## What any_join_distinct_right_table_keys Controls

When `ANY JOIN` encounters multiple right-table rows with the same key, this setting determines which row is returned:

| Value | Behavior |
|---|---|
| `0` (default) | Return the first row encountered for each key (implementation-defined order) |
| `1` | Deduplicate the right table on the join key before joining, ensuring distinct keys |

The practical difference is subtle but matters for reproducibility:

- `0`: ClickHouse returns whichever row it happens to encounter first for a given key. This can vary depending on part order, thread scheduling, and data layout.
- `1`: ClickHouse deduplicates the right table first (keeping one row per key in an unspecified but consistent order), then performs the join. Results are more predictable but still not fully deterministic about *which* duplicate is kept.

## Checking and Setting the Value

```sql
-- Check the current setting
SELECT value, changed
FROM system.settings
WHERE name = 'any_join_distinct_right_table_keys';

-- Set per query
SELECT
    e.user_id,
    e.event_type,
    u.user_name
FROM events AS e
ANY LEFT JOIN users AS u ON e.user_id = u.user_id
SETTINGS any_join_distinct_right_table_keys = 1;
```

In a user profile:

```xml
<clickhouse>
    <profiles>
        <default>
            <any_join_distinct_right_table_keys>1</any_join_distinct_right_table_keys>
        </default>
    </profiles>
</clickhouse>
```

## Practical Examples

### Basic ANY LEFT JOIN

```sql
-- Return the latest user profile for each event (users may have multiple rows)
SELECT
    e.event_date,
    e.event_type,
    u.user_name,
    u.plan_type
FROM events AS e
ANY LEFT JOIN users AS u ON e.user_id = u.user_id
WHERE e.event_date = today()
SETTINGS any_join_distinct_right_table_keys = 1;
```

### ANY INNER JOIN

```sql
-- Include only events for users that exist, one match per event
SELECT
    e.event_id,
    e.event_type,
    u.country
FROM events AS e
ANY INNER JOIN users AS u ON e.user_id = u.user_id
WHERE e.event_date >= today() - 7
SETTINGS any_join_distinct_right_table_keys = 1;
```

### Comparing ANY JOIN to INNER JOIN on Duplicated Right Keys

```sql
-- Setup: right table has duplicate keys
CREATE TABLE dim_products
(
    product_id  UInt64,
    product_name String,
    updated_at  DateTime
)
ENGINE = MergeTree
ORDER BY (product_id, updated_at);

INSERT INTO dim_products VALUES
    (1, 'Widget v1', '2024-01-01 00:00:00'),
    (1, 'Widget v2', '2024-06-01 00:00:00'),
    (2, 'Gadget',    '2024-01-01 00:00:00');

-- INNER JOIN: two rows for product_id=1
SELECT o.order_id, p.product_name
FROM orders AS o
INNER JOIN dim_products AS p ON o.product_id = p.product_id
WHERE o.product_id = 1;
-- Returns: (order_id_1, 'Widget v1'), (order_id_1, 'Widget v2')

-- ANY INNER JOIN: one row for product_id=1
SELECT o.order_id, p.product_name
FROM orders AS o
ANY INNER JOIN dim_products AS p ON o.product_id = p.product_id
WHERE o.product_id = 1
SETTINGS any_join_distinct_right_table_keys = 1;
-- Returns: (order_id_1, 'Widget v1') or (order_id_1, 'Widget v2') - one row
```

## ANY JOIN vs. Using DISTINCT in a Subquery

An alternative to `ANY JOIN` is to deduplicate the right table explicitly before joining:

```sql
-- Explicit deduplication using LIMIT BY
SELECT
    e.event_type,
    u.user_name
FROM events AS e
INNER JOIN (
    SELECT
        user_id,
        argMax(user_name, updated_at) AS user_name
    FROM users
    GROUP BY user_id
) AS u ON e.user_id = u.user_id;
```

The subquery approach using `argMax` is more explicit about which row is kept (the one with the latest `updated_at`). `ANY JOIN` with `any_join_distinct_right_table_keys = 1` does not give you control over which duplicate is retained.

Use the subquery approach when you need deterministic control over which duplicate row is used. Use `ANY JOIN` when any one match per key is acceptable.

## ANY JOIN with Join Tables Engine

The Join table engine stores data in memory pre-indexed for fast lookups. It is designed to work with `ANY JOIN`:

```sql
-- Create a Join table for fast lookups
CREATE TABLE user_lookup
(
    user_id   UInt64,
    user_name String,
    country   String
)
ENGINE = Join(ANY, LEFT, user_id);

-- Populate it
INSERT INTO user_lookup
SELECT user_id, user_name, country FROM users;

-- Fast ANY LEFT JOIN using the Join engine
SELECT
    e.event_type,
    joinGet('user_lookup', 'user_name', e.user_id) AS user_name,
    joinGet('user_lookup', 'country', e.user_id) AS country
FROM events AS e
WHERE e.event_date = today();
```

The Join table engine with `ANY` type automatically handles duplicate keys according to the insertion order: the first-inserted row per key is retained.

## Performance Implications

With `any_join_distinct_right_table_keys = 1`, ClickHouse must deduplicate the right table before building the hash map. For a right table with many rows but relatively few unique keys, this deduplication step can actually reduce the hash map size and improve performance. For right tables that are already unique on the join key, the deduplication adds minimal overhead.

```sql
-- Measure the effect of the setting on a specific query
SELECT count()
FROM events AS e
ANY LEFT JOIN large_dim_table AS d ON e.product_id = d.product_id
SETTINGS
    any_join_distinct_right_table_keys = 1,
    log_comment = 'any_join_dedup_on';
```

## When to Use This Setting

Enable `any_join_distinct_right_table_keys = 1` when:

- Your right table may contain duplicate join keys due to SCD (Slowly Changing Dimension) patterns, audit logs, or data quality issues.
- You want consistent results from `ANY JOIN` across multiple runs of the same query.
- You are migrating queries from a system that guaranteed distinct right-table keys and want ClickHouse to enforce the same behavior.

Leave it at `0` when:

- You are certain the right table always has unique join keys (the default behavior and the deduplication step are equivalent).
- You want maximum performance and are willing to accept that duplicate right-table rows may produce varying results.

## Conclusion

`any_join_distinct_right_table_keys` controls how ClickHouse handles duplicate keys in the right table of an `ANY JOIN`. Set it to `1` for predictable, consistent results when your right table may not have unique join keys. For full control over which duplicate row is kept, use `argMax` aggregation in a subquery instead of relying on `ANY JOIN` semantics. Enable the setting in user profiles where `ANY JOIN` is commonly used with dimension tables that may contain historical or duplicate records.
