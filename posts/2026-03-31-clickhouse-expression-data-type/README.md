# How to Use Expression Data Type in ClickHouse Columns

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Type, Expression, Computed Column

Description: Learn how to define computed columns in ClickHouse using MATERIALIZED and ALIAS expressions to automate derivations at insert or query time.

---

ClickHouse supports computed columns through two expression mechanisms: `MATERIALIZED` and `ALIAS`. Both allow you to define a column whose value is derived from an expression rather than explicitly inserted. `MATERIALIZED` columns are computed at insert time and stored on disk. `ALIAS` columns are computed at query time and never stored. Together they let you encapsulate derivation logic inside the table schema, keeping application insert code simple and query code clean.

## MATERIALIZED Columns

A `MATERIALIZED` column is always computed from its expression when a row is inserted. You cannot provide a value for it directly - ClickHouse ignores any explicitly provided value and always uses the expression.

```sql
CREATE TABLE page_views (
    url          String,
    visited_at   DateTime,

    -- Derived at insert time, stored on disk
    visit_date   Date     MATERIALIZED toDate(visited_at),
    visit_hour   UInt8    MATERIALIZED toHour(visited_at),
    visit_dow    UInt8    MATERIALIZED toDayOfWeek(visited_at),
    url_domain   String   MATERIALIZED domain(url)
) ENGINE = MergeTree()
ORDER BY (url_domain, visited_at);
```

Insert without providing the materialized columns:

```sql
INSERT INTO page_views (url, visited_at) VALUES
('https://example.com/blog/post-1', '2024-06-15 14:32:00'),
('https://docs.example.com/guide',  '2024-06-15 08:10:45'),
('https://example.com/pricing',     '2024-06-16 22:05:12');
```

Query the materialized columns directly - they are already computed and stored:

```sql
SELECT url_domain, visit_date, visit_hour, visit_dow
FROM page_views
ORDER BY visited_at;
-- url_domain=example.com, visit_date=2024-06-15, visit_hour=14, visit_dow=6
```

Because materialized columns are stored, they can be used in `ORDER BY`, `WHERE`, and index key expressions, making them powerful for pre-computing frequently-filtered derivations.

## Using MATERIALIZED for Data Normalization

A common use case is normalizing incoming data at insert time so queries always see clean values:

```sql
CREATE TABLE user_events (
    user_id      UInt64,
    raw_email    String,
    action       String,
    ts           DateTime,

    -- Normalize to lowercase at insert time
    email        String  MATERIALIZED lower(trim(raw_email)),

    -- Extract domain for grouping
    email_domain String  MATERIALIZED domain(lower(trim(raw_email))),

    -- Bucket the hour into time-of-day segments
    time_bucket  String  MATERIALIZED
        multiIf(
            toHour(ts) < 6,  'night',
            toHour(ts) < 12, 'morning',
            toHour(ts) < 18, 'afternoon',
            'evening'
        )
) ENGINE = MergeTree()
ORDER BY (email_domain, ts);

INSERT INTO user_events (user_id, raw_email, action, ts) VALUES
(1, ' Alice@Example.COM ', 'login',    '2024-06-15 09:15:00'),
(2, 'bob@other.net',       'purchase', '2024-06-15 14:32:00'),
(3, ' CAROL@Example.COM',  'logout',   '2024-06-15 22:44:00');

SELECT email, email_domain, time_bucket FROM user_events;
-- email=alice@example.com, email_domain=example.com, time_bucket=morning
```

## ALIAS Columns

An `ALIAS` column is not stored. It is computed fresh each time it appears in a query. This makes it ideal for views or convenience expressions that would be redundant to store.

```sql
CREATE TABLE orders (
    order_id     UInt64,
    subtotal     Float64,
    tax_rate     Float32  DEFAULT 0.08,
    shipping_fee Float64  DEFAULT 5.0,

    -- Computed at query time, not stored
    tax_amount   Float64  ALIAS subtotal * tax_rate,
    total        Float64  ALIAS subtotal + (subtotal * tax_rate) + shipping_fee
) ENGINE = MergeTree()
ORDER BY order_id;

INSERT INTO orders (order_id, subtotal) VALUES
(1, 100.0),
(2, 250.0),
(3, 49.99);

SELECT order_id, subtotal, tax_amount, total FROM orders;
-- order_id=1, subtotal=100.0, tax_amount=8.0, total=113.0
```

Because `ALIAS` columns reference stored columns and recalculate at query time, they always reflect the current stored values - including any updates made via `ALTER TABLE ... UPDATE`.

## Differences Between MATERIALIZED and ALIAS

```sql
CREATE TABLE comparison_demo (
    value_ms   UInt64,

    -- MATERIALIZED: computed at insert, stored, fast to read
    value_sec_stored  Float64  MATERIALIZED value_ms / 1000.0,

    -- ALIAS: computed at query time, not stored, always current
    value_sec_live    Float64  ALIAS value_ms / 1000.0
) ENGINE = MergeTree()
ORDER BY value_ms;
```

| Property | MATERIALIZED | ALIAS |
|----------|-------------|-------|
| Stored on disk | Yes | No |
| Disk space used | Yes | No |
| Available in WHERE | Yes (uses stored value) | Yes (recomputed) |
| Available in ORDER BY | Yes | No |
| Appears in SELECT * | No | No |
| Can be used in primary key | Yes | No |
| Reflects UPDATE changes | No (computed at insert) | Yes (recomputed) |

## Altering Expression Columns

Add, modify, or drop expression columns without rewriting existing data:

```sql
-- Add a new materialized column to an existing table
ALTER TABLE page_views ADD COLUMN url_path String MATERIALIZED path(url);

-- Materialize the column for existing rows (optional, runs as a background mutation)
ALTER TABLE page_views MATERIALIZE COLUMN url_path;

-- Change an alias expression
ALTER TABLE orders MODIFY COLUMN total Float64 ALIAS subtotal * 1.1 + shipping_fee;

-- Remove an expression column
ALTER TABLE orders DROP COLUMN tax_amount;
```

The `MATERIALIZE COLUMN` command backfills the stored values for existing rows using the expression - useful after adding a `MATERIALIZED` column to a populated table.

## Practical Pattern - Pre-computed Partition Keys

A common pattern is to use a `MATERIALIZED` column as a partition key to avoid computing it in every query:

```sql
CREATE TABLE logs (
    message    String,
    level      LowCardinality(String),
    ts         DateTime64(3),
    log_date   Date  MATERIALIZED toDate(ts)
) ENGINE = MergeTree()
PARTITION BY log_date
ORDER BY (level, ts);
```

Partitioning by the materialized `log_date` allows ClickHouse to prune entire partitions for date-range queries without evaluating `toDate(ts)` on every row.

## Summary

ClickHouse `MATERIALIZED` columns compute expressions at insert time and store the result, enabling fast reads and index usage on derived values. `ALIAS` columns compute expressions at query time with no storage cost, always reflecting the current underlying data. Use `MATERIALIZED` for frequently queried derivations like date extraction and domain normalization, and `ALIAS` for convenience calculations you want available without storage overhead.
