# How to Translate PostgreSQL Queries to ClickHouse SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, PostgreSQL, SQL Translation, Migration, Analytics

Description: Convert PostgreSQL SQL patterns to ClickHouse SQL including CTEs, window functions, array operations, and JSONB queries with side-by-side examples.

---

PostgreSQL and ClickHouse share ANSI SQL foundations but differ significantly in mutation semantics, array handling, and JSON support. This guide covers the most common translation patterns.

## CTEs

Both databases support CTEs with `WITH`. ClickHouse materializes CTEs referenced multiple times:

```sql
-- PostgreSQL
WITH monthly AS (
    SELECT date_trunc('month', ts) AS month, sum(amount) AS total
    FROM sales
    GROUP BY 1
)
SELECT month, total, total - lag(total) OVER (ORDER BY month) AS delta
FROM monthly;

-- ClickHouse (toStartOfMonth instead of date_trunc)
WITH monthly AS (
    SELECT toStartOfMonth(ts) AS month, sum(amount) AS total
    FROM sales
    GROUP BY month
)
SELECT month, total, total - lagInFrame(total) OVER (ORDER BY month) AS delta
FROM monthly;
```

## Window Functions

ClickHouse uses `lagInFrame` and `leadInFrame` instead of `lag` and `lead`:

```sql
-- PostgreSQL
SELECT id, value, lag(value, 1) OVER (ORDER BY ts) AS prev_value FROM metrics;

-- ClickHouse
SELECT id, value, lagInFrame(value, 1) OVER (ORDER BY ts) AS prev_value FROM metrics;
```

## Array Operations

```sql
-- PostgreSQL (JSONB array)
SELECT * FROM events WHERE tags @> ARRAY['analytics'];

-- ClickHouse (Array column)
SELECT * FROM events WHERE has(tags, 'analytics');

-- PostgreSQL unnest
SELECT unnest(tags) AS tag FROM events;

-- ClickHouse arrayJoin
SELECT arrayJoin(tags) AS tag FROM events;
```

## JSON Handling

```sql
-- PostgreSQL JSONB
SELECT data->>'user_id', data->'metadata'->>'source'
FROM events;

-- ClickHouse (JSON stored as String)
SELECT
    JSONExtractString(data, 'user_id'),
    JSONExtractString(JSONExtractString(data, 'metadata'), 'source')
FROM events;
```

## UPSERT (INSERT ... ON CONFLICT)

PostgreSQL's upsert has no direct equivalent in ClickHouse. Use `ReplacingMergeTree` instead:

```sql
-- PostgreSQL
INSERT INTO users (id, name) VALUES (1, 'Alice')
ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name;

-- ClickHouse approach: use ReplacingMergeTree and query the latest version
CREATE TABLE users (id UInt64, name String, updated_at DateTime)
ENGINE = ReplacingMergeTree(updated_at) ORDER BY id;

INSERT INTO users VALUES (1, 'Alice', now());
-- Query: SELECT DISTINCT ON (id) id, name FROM users ORDER BY id, updated_at DESC;
```

## Key Incompatibilities

| PostgreSQL | ClickHouse Alternative |
|---|---|
| `SERIAL` / sequences | `generateUUIDv4()` or manual sequences |
| `RETURNING` clause | Not supported; query after insert |
| Full ACID transactions | No multi-statement transactions |
| `LATERAL` joins | Use subqueries or `arrayJoin` |

## Summary

Most PostgreSQL SELECT queries translate to ClickHouse with date function substitutions and `lagInFrame` for window lag functions. Array and JSON handling use ClickHouse-specific functions. Mutations require a different design pattern using ReplacingMergeTree or AggregatingMergeTree.
