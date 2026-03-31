# ClickHouse for MySQL Developers - Key Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MySQL, Migration, OLAP, Columnar Database

Description: A practical guide for MySQL developers learning ClickHouse, covering the key differences in data model, query patterns, and operational behavior.

---

MySQL and ClickHouse serve fundamentally different purposes. MySQL is an OLTP database optimized for fast single-row reads and writes. ClickHouse is an OLAP database optimized for scanning billions of rows in seconds. Understanding these differences helps MySQL developers use ClickHouse effectively.

## Data Storage Model

MySQL stores data row-by-row. ClickHouse stores data column-by-column. When your query only reads 3 columns from a 100-column table, ClickHouse only reads those 3 columns from disk.

```sql
-- MySQL: reads all columns from row storage even for count
SELECT count(*) FROM orders WHERE status = 'shipped';

-- ClickHouse: reads only the 'status' column
SELECT count() FROM orders WHERE status = 'shipped';
```

## No Primary Key Enforcement

ClickHouse `ORDER BY` defines how data is sorted on disk (the sorting key), but it does not enforce uniqueness:

```sql
-- ClickHouse: rows with the same order_id are allowed
CREATE TABLE orders (
    order_id UInt64,
    user_id UInt32,
    amount Decimal(10,2),
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY (user_id, created_at);
```

## INSERT Behavior

ClickHouse is designed for batch inserts, not single-row inserts:

```sql
-- Bad: avoid single-row inserts
INSERT INTO events VALUES (1, 'click', now());

-- Good: batch thousands of rows together
INSERT INTO events VALUES
    (1, 'click', now()),
    (2, 'view', now()),
    -- ... thousands more rows
    (10000, 'purchase', now());
```

## UPDATE and DELETE are Expensive

ClickHouse mutations rewrite entire data parts. Use them sparingly:

```sql
-- Avoid frequent updates - ClickHouse is append-optimized
ALTER TABLE users UPDATE email = 'new@example.com' WHERE user_id = 42;

-- Better approach: use ReplacingMergeTree for upsert patterns
CREATE TABLE users (
    user_id UInt32,
    email String,
    updated_at DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;
```

## NULL Handling

ClickHouse columns are NOT NULL by default. Use `Nullable(Type)` explicitly:

```sql
CREATE TABLE orders (
    order_id UInt64,
    coupon_code Nullable(String),  -- can be NULL
    amount Float64                  -- NOT NULL by default
) ENGINE = MergeTree()
ORDER BY order_id;
```

## JOIN Limitations

ClickHouse JOINs work but are less optimized than MySQL for complex multi-table joins. The recommended pattern is to keep large tables on the left and small tables on the right, or use dictionaries for dimension lookups:

```sql
-- Preferred: large fact table LEFT JOIN small dimension
SELECT o.order_id, u.name
FROM orders o
LEFT JOIN users u ON o.user_id = u.user_id
WHERE o.created_at > now() - INTERVAL 7 DAY;
```

## Summary

MySQL developers transitioning to ClickHouse need to shift from row-oriented thinking to columnar, batch-oriented patterns. ClickHouse excels at aggregate queries over large datasets but is not designed for frequent single-row updates or complex normalized schemas. Embrace append-only data models and materialized views to get the best performance.
