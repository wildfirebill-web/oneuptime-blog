# How to Optimize IN Clause with Large Lists in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, IN Clause, Query Optimization, Performance, Dictionary

Description: Optimize ClickHouse IN clauses with large value lists using subqueries, dictionaries, JOIN rewrites, and bloom filter indexes.

---

Large IN lists in ClickHouse can be slow because the engine must evaluate membership for every row. When the list has thousands of values, alternative approaches are significantly faster.

## Understand the Problem

A large IN list with literals is compiled at query parse time and evaluated per row:

```sql
-- This can be slow for thousands of values
SELECT count() FROM events
WHERE user_id IN (1001, 1002, 1003, ..., 50000);
```

## Use a Subquery Instead of Literal List

Move the values to a table and use a subquery:

```sql
-- Store the filter set
CREATE TABLE target_users (
  user_id UInt64
) ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO target_users VALUES (1001), (1002), ..., (50000);

-- Query with subquery
SELECT count()
FROM events
WHERE user_id IN (SELECT user_id FROM target_users);
```

ClickHouse will build a hash set from the subquery and use it efficiently.

## Rewrite as a JOIN

For very large filter sets, an explicit JOIN can be faster and more transparent:

```sql
SELECT count()
FROM events e
INNER JOIN target_users t ON e.user_id = t.user_id;
```

Place the smaller table on the right side. ClickHouse loads the right table into memory as a hash table.

## Use a Dictionary for Membership Tests

Dictionaries provide the fastest membership lookup with automatic refresh:

```sql
CREATE DICTIONARY target_user_dict (
  user_id UInt64
) PRIMARY KEY user_id
SOURCE(CLICKHOUSE(query 'SELECT user_id FROM target_users'))
LAYOUT(HASHED())
LIFETIME(MIN 60 MAX 300);

-- Use dictHas for O(1) lookup
SELECT count()
FROM events
WHERE dictHas('target_user_dict', user_id);
```

## Use Bloom Filter Index

For large IN queries on columns in MergeTree tables, add a bloom filter skip index:

```sql
ALTER TABLE events
  ADD INDEX user_id_bf user_id TYPE bloom_filter(0.01) GRANULARITY 1;

ALTER TABLE events MATERIALIZE INDEX user_id_bf;
```

This helps skip data parts that cannot contain any of the IN values.

## Batch Large IN Lists

If you must use literal IN lists (e.g., from application code), batch the queries:

```sql
-- Instead of one query with 100k values,
-- run 10 queries with 10k values each and UNION ALL
SELECT count() FROM events WHERE user_id IN (1..10000)
UNION ALL
SELECT count() FROM events WHERE user_id IN (10001..20000);
```

## Verify with EXPLAIN

Check if the IN clause is using a hash set or doing a scan:

```sql
EXPLAIN PLAN
SELECT count() FROM events
WHERE user_id IN (SELECT user_id FROM target_users);
```

## Summary

Optimizing large IN lists in ClickHouse means moving literal value lists to tables and using subqueries, rewriting as INNER JOINs, creating dictionaries for O(1) membership tests, or adding bloom filter skip indexes. Each approach reduces per-row evaluation cost, with dictionaries offering the best performance for frequently reused filter sets.
