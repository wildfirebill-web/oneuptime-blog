# How to Optimize Array Operations in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array, Performance, ARRAY JOIN, arrayMap, Query Optimization

Description: Optimize ClickHouse array operations using arrayFilter, arrayMap, ARRAY JOIN, and proper column types for faster analytical queries.

---

ClickHouse has powerful array support but array operations can be slow if used carelessly. Understanding which functions are vectorized and how ARRAY JOIN works helps you write efficient queries.

## Use Typed Arrays

Always specify the element type explicitly. Untyped or mismatched types cause implicit conversions:

```sql
-- Slow: type inference and possible implicit casting
CREATE TABLE events (
  tags Array(String),
  scores Array(Float64)
) ENGINE = MergeTree() ORDER BY tuple();

-- Fast: use specific low-overhead types
CREATE TABLE events_typed (
  tags Array(LowCardinality(String)),
  scores Array(Float32)  -- Float32 if precision allows
) ENGINE = MergeTree() ORDER BY tuple();
```

## Filter with arrayExists Instead of ARRAY JOIN

For checking membership, `arrayExists` is faster than unnesting with `ARRAY JOIN`:

```sql
-- Slow: unnests every element, then filters
SELECT count()
FROM events
ARRAY JOIN tags
WHERE tags = 'purchase';

-- Fast: checks membership without unnesting
SELECT count()
FROM events
WHERE arrayExists(x -> x = 'purchase', tags);

-- Even simpler
SELECT count()
FROM events
WHERE has(tags, 'purchase');
```

## Use arrayFilter and arrayMap for Bulk Operations

Process arrays without row-by-row ARRAY JOIN:

```sql
-- Sum only positive scores per row
SELECT user_id, arraySum(arrayFilter(x -> x > 0, scores)) AS positive_sum
FROM events;

-- Double all scores
SELECT user_id, arrayMap(x -> x * 2, scores) AS doubled_scores
FROM events;
```

These are vectorized and much faster than looping.

## Avoid arrayJoin() in SELECT with ARRAY JOIN

`arrayJoin()` function in the SELECT list and the `ARRAY JOIN` clause do different things and the function version is generally slower:

```sql
-- Use ARRAY JOIN clause
SELECT tag, count()
FROM events
ARRAY JOIN tags AS tag
GROUP BY tag;

-- Avoid: arrayJoin() in SELECT is not vectorized
SELECT arrayJoin(tags) AS tag, count()
FROM events
GROUP BY tag;
```

## Use ARRAY JOIN with Multiple Arrays

When joining multiple arrays of the same length, use a single ARRAY JOIN:

```sql
-- Correct: joins paired arrays element-by-element
SELECT tag, score
FROM events
ARRAY JOIN tags AS tag, scores AS score;
```

## Index Arrays with bloom_filter

Add a bloom filter index to array columns to speed up `has()` lookups:

```sql
ALTER TABLE events
  ADD INDEX tags_bf tags TYPE bloom_filter(0.01) GRANULARITY 1;

ALTER TABLE events MATERIALIZE INDEX tags_bf;

-- Now has() can skip granules
SELECT count() FROM events WHERE has(tags, 'purchase');
```

## Summary

ClickHouse array operations are optimized by using typed arrays with `LowCardinality` where applicable, preferring `has()` and `arrayExists()` over `ARRAY JOIN` for membership tests, using `arrayFilter`/`arrayMap` for vectorized transformations, and adding bloom filter skip indexes on array columns used in `has()` queries. These changes reduce both I/O and CPU overhead significantly.
