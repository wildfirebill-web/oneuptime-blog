# How to Use values() Table Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Table Function, SQL, Database, Query

Description: Learn how to use the values() table function in ClickHouse to generate inline row sets directly in SQL queries, with practical examples for testing, seeding, and data transformation.

---

The `values()` table function in ClickHouse lets you define an inline set of rows directly inside a SQL query. It is one of the simplest ways to work with ad-hoc data without creating a physical table, making it ideal for quick tests, seed data, and transformations.

## What Is the values() Table Function?

`values()` generates a temporary, in-memory result set from literal values you supply in the query itself. Think of it as SQL's equivalent of an array literal - you describe the schema and the data in one place.

```sql
SELECT *
FROM values('id UInt32, name String', (1, 'Alice'), (2, 'Bob'), (3, 'Charlie'));
```

```text
┌─id─┬─name────┐
│  1 │ Alice   │
│  2 │ Bob     │
│  3 │ Charlie │
└────┴─────────┘
```

## Basic Syntax

```sql
values('<column_name> <type>[, ...]', (v1, v2, ...), (v1, v2, ...), ...)
```

The first argument is a comma-separated schema string. Each subsequent argument is a tuple of values matching that schema.

## Using values() in INSERT Statements

A common use case is seeding a real table with known test data:

```sql
CREATE TABLE products
(
    product_id UInt32,
    name       String,
    price      Float64,
    in_stock   UInt8
)
ENGINE = MergeTree()
ORDER BY product_id;

INSERT INTO products
SELECT *
FROM values(
    'product_id UInt32, name String, price Float64, in_stock UInt8',
    (1, 'Laptop',  999.99, 1),
    (2, 'Monitor', 349.50, 1),
    (3, 'Keyboard', 89.00, 0),
    (4, 'Mouse',    45.00, 1)
);
```

## Filtering Inline Data

You can apply a `WHERE` clause directly to a `values()` result:

```sql
SELECT *
FROM values('product_id UInt32, name String, price Float64', (1, 'Laptop', 999.99), (2, 'Monitor', 349.50), (3, 'Keyboard', 89.00))
WHERE price > 100;
```

```text
┌─product_id─┬─name────┬──price─┐
│          1 │ Laptop  │ 999.99 │
│          2 │ Monitor │  349.5 │
└────────────┴─────────┴────────┘
```

## Joining values() with a Real Table

This is especially useful when you need to enrich live data with a small static mapping:

```sql
-- Map numeric status codes to human-readable labels on the fly
SELECT
    e.event_id,
    e.status_code,
    m.label
FROM events AS e
JOIN values(
    'status_code UInt8, label String',
    (0, 'Unknown'),
    (1, 'Active'),
    (2, 'Inactive'),
    (3, 'Deleted')
) AS m ON e.status_code = m.status_code;
```

## Using values() for Quick Type Tests

When you want to verify how ClickHouse handles a specific data type or function, `values()` saves you from creating a throw-away table:

```sql
-- Test date arithmetic without a table
SELECT
    event_date,
    toStartOfMonth(event_date) AS month_start,
    toDayOfWeek(event_date)    AS day_num
FROM values(
    'event_date Date',
    ('2026-01-15'),
    ('2026-02-28'),
    ('2026-03-01')
);
```

## Generating Lookup Tables Inline

```sql
-- Classify temperature readings inline
SELECT
    reading,
    CASE
        WHEN reading < 0   THEN 'Freezing'
        WHEN reading < 15  THEN 'Cold'
        WHEN reading < 25  THEN 'Comfortable'
        ELSE                    'Hot'
    END AS classification
FROM values('reading Float32', (-5), (10), (22), (35), (40));
```

## Nested Data Types in values()

`values()` supports complex types including arrays and tuples:

```sql
SELECT *
FROM values(
    'user_id UInt32, scores Array(UInt8), meta Tuple(String, UInt8)',
    (1, [90, 85, 92], ('Alice', 28)),
    (2, [70, 80, 75], ('Bob',   32))
);
```

## Using values() in Subqueries

```sql
SELECT
    orders.order_id,
    orders.amount,
    regions.region_name
FROM orders
JOIN (
    SELECT *
    FROM values(
        'region_code String, region_name String',
        ('US', 'United States'),
        ('EU', 'Europe'),
        ('AP', 'Asia Pacific')
    )
) AS regions ON orders.region_code = regions.region_code;
```

## Performance Notes

- `values()` data lives entirely in memory. Keep the number of rows small (hundreds to low thousands). For large data sets, write the data to a real table first.
- There is no disk I/O, so `values()` is very fast for small data.
- The query planner treats `values()` the same as any other table function; you can apply all standard SQL clauses to it.

## Common Mistakes

**Wrong tuple count** - every tuple must have exactly as many elements as the schema has columns:

```sql
-- This will fail: schema has 3 columns but tuple has 2 values
SELECT *
FROM values('a UInt32, b String, c Float64', (1, 'hello'));
```

**Type mismatches** - ClickHouse performs implicit casting, but be careful with nullable vs non-nullable types:

```sql
-- Explicit cast when mixing nullable and non-nullable contexts
SELECT *
FROM values(
    'id UInt32, value Nullable(Float64)',
    (1, 3.14),
    (2, NULL)
);
```

## Summary

The `values()` table function is a lightweight tool for working with inline data in ClickHouse. Key takeaways:

- Use it for rapid prototyping and testing queries without creating tables.
- Use it as a seed source in `INSERT INTO ... SELECT` workflows.
- Join it with real tables to apply small static mappings.
- Keep row counts small - it is an in-memory construct.
