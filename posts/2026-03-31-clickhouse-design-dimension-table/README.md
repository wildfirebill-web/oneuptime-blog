# How to Design a Dimension Table in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dimension Table, Schema Design, Dictionary, Data Warehouse, Lookup

Description: Learn how to design efficient dimension tables in ClickHouse, including dictionary-based lookups that eliminate join overhead in analytical queries.

---

Dimension tables provide descriptive context to fact tables. In ClickHouse, dimension tables can be stored as regular MergeTree tables or as dictionaries, which offer significantly faster lookups at query time.

## Standard Dimension Table

For a product dimension:

```sql
CREATE TABLE dim_product (
    product_id UInt32,
    sku String,
    name String,
    category LowCardinality(String),
    sub_category LowCardinality(String),
    brand LowCardinality(String),
    supplier_id UInt32,
    unit_cost Decimal64(2),
    is_active Bool DEFAULT true,
    created_at Date,
    updated_at DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY product_id;
```

Using `ReplacingMergeTree` allows you to update dimension records by reinserting with a newer `updated_at`. Query with FINAL for deduplication:

```sql
SELECT * FROM dim_product FINAL WHERE is_active = true;
```

## Converting to a Dictionary

For frequently-joined dimensions, a dictionary eliminates JOIN overhead entirely:

```sql
CREATE DICTIONARY product_dict (
    product_id UInt32,
    name String,
    category String,
    brand String
) PRIMARY KEY product_id
SOURCE(CLICKHOUSE(TABLE 'dim_product' WHERE 'is_active = 1'))
LAYOUT(HASHED())
LIFETIME(MIN 3600 MAX 7200);
```

Now use `dictGet` instead of JOIN:

```sql
SELECT
    f.order_id,
    dictGet('product_dict', 'category', f.product_id) AS category,
    dictGet('product_dict', 'brand', f.product_id) AS brand,
    sum(f.revenue) AS revenue
FROM fact_orders f
WHERE toDate(f.ordered_at) = today()
GROUP BY f.order_id, category, brand;
```

## Hierarchical Dimension

For dimensions with parent-child relationships:

```sql
CREATE TABLE dim_geography (
    geo_id UInt32,
    geo_type LowCardinality(String),  -- country, state, city
    name String,
    parent_id UInt32 DEFAULT 0,
    country_code LowCardinality(FixedString(2)),
    timezone String,
    latitude Float32,
    longitude Float32
) ENGINE = MergeTree()
ORDER BY (geo_type, geo_id);

-- Hierarchy: city -> state -> country
-- Use dictGetHierarchy for traversal
```

## Date Dimension

A date dimension provides pre-computed calendar attributes:

```sql
CREATE TABLE dim_date (
    date_key UInt32,        -- YYYYMMDD
    date Date,
    year UInt16,
    year_quarter String,    -- 2025-Q1
    quarter UInt8,
    month UInt8,
    month_name LowCardinality(String),
    week_of_year UInt8,
    day_of_month UInt8,
    day_of_week UInt8,
    day_name LowCardinality(String),
    is_weekend Bool,
    is_holiday Bool DEFAULT false,
    fiscal_year UInt16,
    fiscal_quarter UInt8
) ENGINE = MergeTree()
ORDER BY date_key;

-- Populate for 10 years
INSERT INTO dim_date
SELECT
    toUInt32(formatDateTime(date, '%Y%m%d')) AS date_key,
    date,
    toYear(date) AS year,
    concat(toString(toYear(date)), '-Q', toString(toQuarter(date))) AS year_quarter,
    toQuarter(date) AS quarter,
    toMonth(date) AS month,
    formatDateTime(date, '%B') AS month_name,
    toISOWeek(date) AS week_of_year,
    toDayOfMonth(date) AS day_of_month,
    toDayOfWeek(date) AS day_of_week,
    formatDateTime(date, '%A') AS day_name,
    toDayOfWeek(date) IN (6, 7) AS is_weekend,
    false AS is_holiday,
    toYear(date) AS fiscal_year,
    toQuarter(date) AS fiscal_quarter
FROM (SELECT today() - number AS date FROM numbers(3650));
```

## Dimension Table Best Practices

```text
1. Use ReplacingMergeTree for mutable dimensions
2. Convert small, frequently-joined dimensions to dictionaries
3. Use LowCardinality for repeated string columns
4. Pre-compute hierarchy levels into flat columns for performance
5. Keep date dimensions as regular tables or dictionaries
```

## Summary

Dimension tables in ClickHouse work best when converted to dictionaries for small, frequently-joined dimensions, using `ReplacingMergeTree` for mutable records, and pre-computing hierarchical relationships into flat columns. Dictionaries with `HASHED` layout provide O(1) lookups that completely eliminate JOIN overhead.
