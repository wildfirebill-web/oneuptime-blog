# How to Use map() to Create Maps in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Map Function, Data Type, Query Optimization

Description: Learn how to use map() to create Map columns and inline key-value structures in ClickHouse queries, with practical examples for storing and querying structured data.

---

The `map()` function is the primary way to create Map values in ClickHouse. A Map is a data type that stores key-value pairs where all keys share the same type and all values share the same type. Maps are particularly useful for storing semi-structured data such as event metadata, configuration settings, or any collection of named attributes that vary per row.

## Basic map() Syntax

The `map()` function accepts alternating key and value arguments. The number of arguments must be even. All keys must be of the same type, and all values must be of the same type.

```text
map(key1, value1, key2, value2, ...)
```

The simplest way to explore `map()` is directly in a SELECT statement without any table.

```sql
SELECT map('color', 'red', 'size', 'large', 'material', 'cotton') AS product_attributes;
```

This returns a single Map(String, String) value containing three entries.

## Creating a Table with a Map Column

Map columns are defined in a CREATE TABLE statement using the `Map(KeyType, ValueType)` syntax. This is useful when you have variable sets of attributes per row and want to avoid creating many sparse nullable columns.

```sql
CREATE TABLE product_events
(
    event_id     UInt64,
    event_type   String,
    occurred_at  DateTime,
    attributes   Map(String, String)
)
ENGINE = MergeTree
ORDER BY (event_type, occurred_at);
```

## Inserting Map Data

When inserting rows, you provide map literals using the `map()` function or the `{key: value}` literal syntax. Both forms are equivalent.

```sql
INSERT INTO product_events VALUES
(1, 'view',     '2024-01-15 10:00:00', map('sku', 'ABC-001', 'category', 'apparel', 'color', 'blue')),
(2, 'add_cart', '2024-01-15 10:05:00', map('sku', 'ABC-001', 'category', 'apparel', 'qty', '2')),
(3, 'purchase', '2024-01-15 10:12:00', map('sku', 'ABC-001', 'category', 'apparel', 'price', '29.99', 'currency', 'USD')),
(4, 'view',     '2024-01-15 11:00:00', map('sku', 'XYZ-042', 'category', 'footwear'));
```

## Accessing Map Values by Key

Use square bracket notation to retrieve the value for a specific key. If the key does not exist, ClickHouse returns the default value for the value type (empty string for String, 0 for numeric types).

```sql
SELECT
    event_id,
    event_type,
    attributes['sku']      AS sku,
    attributes['category'] AS category,
    attributes['price']    AS price
FROM product_events;
```

## Creating Maps Inline in Queries

You can use `map()` directly inside queries to build temporary map structures for comparison, joining, or transformation purposes. This is useful when constructing lookup maps on the fly.

```sql
SELECT
    event_id,
    event_type,
    attributes,
    map(
        'view',     'Page View',
        'add_cart', 'Added to Cart',
        'purchase', 'Completed Purchase'
    )[event_type] AS event_label
FROM product_events;
```

## Using Maps with Numeric Values

Maps are not limited to string values. You can store numeric data, which is useful for counters, scores, or measurements per named dimension.

```sql
CREATE TABLE user_scores
(
    user_id    UInt64,
    week_start Date,
    scores     Map(String, Float64)
)
ENGINE = MergeTree
ORDER BY (user_id, week_start);

INSERT INTO user_scores VALUES
(101, '2024-01-08', map('accuracy', 0.92, 'speed', 0.78, 'consistency', 0.85)),
(102, '2024-01-08', map('accuracy', 0.88, 'speed', 0.95, 'consistency', 0.71)),
(103, '2024-01-08', map('accuracy', 0.75, 'speed', 0.80));
```

Retrieve and compute on numeric map values directly in SELECT expressions.

```sql
SELECT
    user_id,
    scores['accuracy']    AS accuracy,
    scores['speed']       AS speed,
    scores['consistency'] AS consistency,
    (scores['accuracy'] + scores['speed'] + scores['consistency']) / 3 AS avg_score
FROM user_scores
ORDER BY avg_score DESC;
```

## Building Maps from Query Results

You can construct a Map dynamically using `map()` with computed expressions as keys or values, useful for creating row-level summaries.

```sql
SELECT
    event_type,
    map(
        'total',    count(),
        'unique_skus', uniq(attributes['sku'])
    ) AS summary
FROM product_events
GROUP BY event_type;
```

## Nested Map Access Pattern

When combining `map()` with conditional expressions, you can build structured results that preserve semantic meaning without requiring schema changes.

```sql
SELECT
    event_id,
    map(
        'raw_type',   event_type,
        'sku',        attributes['sku'],
        'is_revenue', toString(event_type = 'purchase')
    ) AS enriched
FROM product_events;
```

## Summary

The `map()` function in ClickHouse provides a flexible way to store and query key-value data without requiring a fixed schema. Use it to create Map columns for variable attribute sets, build inline maps for lookups and transformations, and store numeric or string metrics under named keys. Map columns work well for event metadata, configuration storage, and any domain where the set of attributes varies by row. Access individual values with bracket notation, and combine `map()` with other map functions like `mapKeys()`, `mapValues()`, and `mapContains()` for richer processing.
