# Why You Should Avoid Large JOINs in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JOIN, Query Optimization, Performance, Best Practice

Description: Explains why large JOIN operations are expensive in ClickHouse and what alternatives - such as dictionaries and denormalization - work better at scale.

---

## How JOINs Work in ClickHouse

ClickHouse supports JOIN but is architecturally optimized for single-table scans, not multi-table joins. For most JOIN algorithms, the right-hand table (the smaller side) is loaded entirely into memory as a hash table. If the right table is large, this exhausts memory and slows queries significantly.

## The Default: Hash Join

```sql
-- The right table (users, ~1M rows) is loaded into memory
SELECT e.user_id, e.event_type, u.country
FROM events AS e
JOIN users AS u ON e.user_id = u.user_id
WHERE e.event_time >= today();
```

If `users` is 10GB, this query loads 10GB into memory per query execution. Running it concurrently across multiple users multiplies the memory pressure.

## What EXPLAIN Reveals

```sql
EXPLAIN PIPELINE
SELECT e.user_id, u.country
FROM events AS e
JOIN users AS u ON e.user_id = u.user_id
WHERE e.event_time >= today();
```

Look for `HashJoin` in the output. If the right side is large, consider alternatives.

## Solution 1: Use Dictionaries for Dimension Lookups

Dictionaries load lookup data once and keep it in shared memory, unlike JOINs that reload per query.

```sql
-- Define a dictionary
CREATE DICTIONARY user_country_dict (
  user_id UInt64,
  country String
)
PRIMARY KEY user_id
SOURCE(CLICKHOUSE(TABLE 'users' DB 'analytics'))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 600);

-- Use it in queries - no JOIN needed
SELECT event_type, dictGet('user_country_dict', 'country', user_id) AS country
FROM events
WHERE event_time >= today();
```

## Solution 2: Denormalize at Ingestion Time

For immutable dimension data, store enriched columns directly in the fact table.

```sql
-- Enriched events table - no join needed at query time
CREATE TABLE events_enriched (
  event_id    UInt64,
  user_id     UInt64,
  country     String,        -- copied from users at insert time
  event_type  String,
  event_time  DateTime
) ENGINE = MergeTree()
ORDER BY (user_id, event_time);
```

## Solution 3: Force Smaller Join Side

If a JOIN is unavoidable, filter the right table aggressively before joining.

```sql
-- Pre-filter to reduce memory footprint
SELECT e.user_id, u.country
FROM events AS e
JOIN (
  SELECT user_id, country FROM users WHERE country = 'US'
) AS u ON e.user_id = u.user_id
WHERE e.event_time >= today();
```

## Solution 4: Change JOIN Algorithm Setting

For specific cases, partial merge join uses less memory at the cost of speed:

```sql
SET join_algorithm = 'partial_merge';

SELECT e.user_id, u.country
FROM events AS e
JOIN users AS u ON e.user_id = u.user_id;
```

## Summary

Large JOINs in ClickHouse load the right-side table entirely into memory, which is costly for dimension tables over a few hundred MB. Use dictionaries for frequently accessed dimension lookups, denormalize data at ingestion for static attributes, and filter join inputs aggressively when joins are unavoidable. For OLAP workloads, these patterns outperform normalized schemas with multi-table joins.
