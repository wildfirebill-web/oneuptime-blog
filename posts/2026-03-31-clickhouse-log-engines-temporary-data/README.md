# How to Use Log Engines for Temporary Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Log Engine, TinyLog, StripeLog, Temporary Table, ETL, SQL

Description: Learn how to use ClickHouse Log family engines for temporary data, ETL staging, and intermediate results where lightweight storage without index overhead is needed.

---

ClickHouse Log family engines (TinyLog, StripeLog, Log) are ideal for temporary data scenarios: ETL staging tables, intermediate query results, test fixtures, and short-lived lookup tables. They write data immediately without background compaction, making them fast for small-volume write-once, read-few patterns.

## Temporary Tables with ENGINE = Memory

For truly temporary data that should not persist beyond the session:

```sql
CREATE TEMPORARY TABLE session_temp (
    id    UInt32,
    value String
);
```

Temporary tables use the Memory engine and are session-scoped. For persistent-but-temporary data across sessions, Log engines are more appropriate.

## ETL Staging with StripeLog

```sql
-- Stage raw CSV import before transformation
CREATE TABLE staging_orders (
    raw_order_id String,
    raw_amount   String,
    raw_date     String,
    raw_customer String
) ENGINE = StripeLog;

-- Load raw data
INSERT INTO staging_orders
SELECT * FROM s3('s3://bucket/raw/orders/*.csv', 'CSVWithNames');

-- Transform and load into production table
INSERT INTO orders (order_id, amount, order_date, customer_id)
SELECT
    toUInt64(raw_order_id),
    toFloat64(raw_amount),
    toDate(raw_date),
    toUInt64(raw_customer)
FROM staging_orders
WHERE raw_order_id != '' AND raw_amount != '';

-- Clean up staging
TRUNCATE TABLE staging_orders;
```

## Intermediate Aggregation Results

```sql
-- Store a complex aggregation result for reuse
CREATE TABLE daily_summary (
    day          Date,
    product_id   UInt32,
    total_revenue Float64,
    order_count  UInt64
) ENGINE = Log;

INSERT INTO daily_summary
SELECT
    toDate(created_at) AS day,
    product_id,
    sum(amount)  AS total_revenue,
    count()      AS order_count
FROM orders
WHERE toDate(created_at) = yesterday()
GROUP BY day, product_id;

-- Use the pre-computed result
SELECT product_id, total_revenue
FROM daily_summary
ORDER BY total_revenue DESC
LIMIT 10;

-- Cleanup
DROP TABLE daily_summary;
```

## Test Fixture Tables

```sql
-- Lightweight test data for integration tests
CREATE TABLE test_users (
    user_id UInt32,
    name    String,
    plan    String
) ENGINE = TinyLog;

INSERT INTO test_users VALUES
    (1, 'alice', 'pro'),
    (2, 'bob', 'free'),
    (3, 'carol', 'enterprise');

-- Run test queries
SELECT count() FROM test_users WHERE plan = 'pro';

-- Teardown
DROP TABLE test_users;
```

## Lookup Tables for JOINs

```sql
-- Small reference table: too small for MergeTree overhead
CREATE TABLE http_status_labels (
    code  UInt16,
    label String
) ENGINE = Log;

INSERT INTO http_status_labels VALUES
    (200, 'OK'), (201, 'Created'), (400, 'Bad Request'),
    (401, 'Unauthorized'), (403, 'Forbidden'), (404, 'Not Found'),
    (500, 'Server Error'), (503, 'Unavailable');

-- Use in query
SELECT
    r.path,
    l.label AS status_label,
    count() AS requests
FROM request_log AS r
LEFT JOIN http_status_labels AS l ON r.status_code = l.code
GROUP BY r.path, status_label
ORDER BY requests DESC;
```

## Choosing Between Log Variants for Temp Data

```text
Scenario                             Engine
< 100 rows, no concurrency           TinyLog
100 - 1M rows, single-session        StripeLog or Log
Multi-reader temp table              Log or StripeLog
Truly session-scoped                 TEMPORARY TABLE (Memory)
```

## Cleanup Patterns

```sql
-- Truncate to clear data without dropping table
TRUNCATE TABLE staging_orders;

-- Drop when fully done
DROP TABLE IF EXISTS daily_summary;
```

## Summary

Log family engines (TinyLog, StripeLog, Log) are the right choice for temporary ETL staging tables, intermediate aggregation storage, test fixtures, and small lookup tables in ClickHouse. They have zero background compaction overhead, immediate write semantics, and simple on-disk layouts. Use StripeLog or Log (not TinyLog) when concurrent reads are needed; use ClickHouse's native TEMPORARY TABLE syntax for session-scoped data that should never persist to disk.
