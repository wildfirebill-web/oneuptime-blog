# How to Create Complex Key Dictionaries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, Complex Key, Lookup, Data Modeling

Description: Learn how to create complex key dictionaries in ClickHouse using composite keys to enable fast multi-column lookups and enrichment queries.

---

Complex key dictionaries in ClickHouse allow you to look up values using more than one column as the key. This is useful when a single attribute is not enough to uniquely identify a dimension record - for example, enriching events by both country code and product ID together.

## What Are Complex Key Dictionaries

A complex key dictionary uses a tuple of columns as the primary key, unlike flat dictionaries which use a single integer or string. ClickHouse supports complex keys for the `hashed`, `cache`, and `direct` layout types.

## Defining a Complex Key Dictionary

Use the `PRIMARY KEY` clause with multiple columns inside the `CREATE DICTIONARY` statement:

```sql
CREATE DICTIONARY product_region_prices
(
    country_code String,
    product_id   UInt64,
    price        Float64
)
PRIMARY KEY country_code, product_id
SOURCE(CLICKHOUSE(
    HOST 'localhost' PORT 9000
    USER 'default' PASSWORD ''
    DB 'sales' TABLE 'region_prices'
))
LAYOUT(COMPLEX_KEY_HASHED())
LIFETIME(MIN 300 MAX 600);
```

The `COMPLEX_KEY_HASHED` layout builds an in-memory hash map keyed on the tuple `(country_code, product_id)`.

## Querying a Complex Key Dictionary

Use `dictGet` with a tuple as the key argument:

```sql
SELECT
    event_id,
    dictGet('product_region_prices', 'price', (country_code, product_id)) AS unit_price
FROM events
LIMIT 10;
```

Use `dictHas` to check existence:

```sql
SELECT dictHas('product_region_prices', ('US', 42)) AS exists;
```

## Available Layouts for Complex Keys

- `COMPLEX_KEY_HASHED` - fast in-memory hash map, good for most use cases
- `COMPLEX_KEY_CACHE` - size-bounded cache, suitable for large dictionaries that do not fit in RAM
- `COMPLEX_KEY_DIRECT` - no caching; reads from the source on each lookup

```sql
-- Using cache layout with a size limit
LAYOUT(COMPLEX_KEY_CACHE(SIZE_IN_CELLS 1000000))
```

## Source Options

Complex key dictionaries support the same sources as flat dictionaries: ClickHouse tables, MySQL, PostgreSQL, HTTP, and local files.

```sql
SOURCE(MYSQL(
    HOST 'mysql-host' PORT 3306
    USER 'reader' PASSWORD 'secret'
    DB 'catalog' TABLE 'product_region'
))
```

## Checking Dictionary Status

```sql
SELECT name, type, status, element_count
FROM system.dictionaries
WHERE name = 'product_region_prices';
```

## Practical Example - Order Enrichment

```sql
SELECT
    order_id,
    dictGet('product_region_prices', 'price', (ship_country, item_id)) AS lookup_price,
    quantity * dictGet('product_region_prices', 'price', (ship_country, item_id)) AS line_total
FROM orders
WHERE event_date = today();
```

## Summary

Complex key dictionaries extend the standard dictionary feature to multi-column lookups. By defining composite primary keys and using the `COMPLEX_KEY_HASHED` or `COMPLEX_KEY_CACHE` layout, you can enrich your ClickHouse queries with region-product or similar multi-dimensional data efficiently without expensive JOIN operations at query time.
