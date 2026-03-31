# Common ClickHouse JOIN Mistakes and How to Fix Them

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JOIN, Query Optimization, Performance, Distributed Query

Description: Avoid the most common ClickHouse JOIN mistakes that cause OOM errors, slow queries, and wrong results in analytical workloads.

---

JOINs in ClickHouse behave differently from traditional RDBMS. Understanding these differences prevents the most painful performance and correctness issues.

## Mistake 1: Putting the Large Table on the Right Side

ClickHouse loads the right-side table into memory as a hash table. Putting a large table on the right causes OOM errors.

```sql
-- Wrong: large fact table on the right
SELECT e.user_id, u.name
FROM dim_users u
JOIN fact_events e ON u.id = e.user_id;

-- Correct: large table on the left, small dimension on the right
SELECT e.user_id, u.name
FROM fact_events e
JOIN dim_users u ON e.user_id = u.id;
```

The left table is streamed; the right table is loaded into a hash map. Always put the smaller lookup table on the right.

## Mistake 2: Using JOIN on Distributed Tables Without GLOBAL

On a distributed cluster, a regular JOIN executes on each shard independently. The right-side subquery is sent to every shard, causing N-fold query fan-out.

```sql
-- Wrong on distributed tables: right side runs on each shard
SELECT e.id, u.name FROM dist_events e JOIN dist_users u ON e.user_id = u.id;

-- Correct: GLOBAL JOIN fetches right side once and broadcasts it
SELECT e.id, u.name FROM dist_events e GLOBAL JOIN dist_users u ON e.user_id = u.id;
```

Use `GLOBAL IN` and `GLOBAL JOIN` whenever the right side is a remote or distributed table.

## Mistake 3: Forgetting join_use_nulls for LEFT JOIN Behavior

By default, ClickHouse fills unmatched rows with default values (0, empty string) rather than NULL. This can cause incorrect aggregations.

```sql
SET join_use_nulls = 1;
SELECT e.id, u.name
FROM events e
LEFT JOIN users u ON e.user_id = u.id
WHERE u.name IS NULL;  -- finds events with no matching user
```

Without `join_use_nulls = 1`, `u.name IS NULL` would never match because unmatched rows get an empty string.

## Mistake 4: Joining on High-Cardinality String Columns

Hashing large strings is expensive. Use integer surrogate keys for JOIN conditions whenever possible, or cast string UUIDs to `FixedString(16)`.

```sql
-- Use integer keys for faster hashing
SELECT e.id, u.name
FROM events e
JOIN users u ON e.user_id_int = u.id_int;
```

## Mistake 5: Not Using Dictionary JOIN for Repeated Lookups

When the same small dimension table is joined in many queries, load it as a dictionary to avoid repeated hash table construction.

```sql
CREATE DICTIONARY dim_users_dict (id UInt64, name String)
PRIMARY KEY id
SOURCE(CLICKHOUSE(TABLE 'dim_users'))
LAYOUT(HASHED())
LIFETIME(300);

SELECT dictGet('dim_users_dict', 'name', toUInt64(user_id)) AS name
FROM events;
```

Dictionary lookups are faster than hash joins for small, frequently-accessed tables.

## Summary

ClickHouse JOINs require putting the large table on the left, using `GLOBAL JOIN` on distributed tables, enabling `join_use_nulls` for correct NULL semantics, and replacing repeated small table joins with dictionaries. These changes can reduce query time from minutes to seconds on analytical workloads.
