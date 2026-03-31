# How to Load Dictionaries from PostgreSQL Sources in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, PostgreSQL, Integration, Data Loading

Description: Learn how to configure ClickHouse dictionaries to load dimension data from a PostgreSQL database, enabling fast in-memory enrichment from your operational store.

---

ClickHouse supports PostgreSQL as a native dictionary source. This allows you to enrich ClickHouse queries with dimension data stored in your PostgreSQL operational database without a separate ETL pipeline.

## Creating a PostgreSQL-Backed Dictionary

```sql
CREATE DICTIONARY product_dict
(
    product_id  UInt64,
    sku         String,
    category    String,
    base_price  Float64
)
PRIMARY KEY product_id
SOURCE(POSTGRESQL(
    HOST     'pg.example.com'
    PORT     5432
    USER     'clickhouse_reader'
    PASSWORD 'secret'
    DB       'shop'
    TABLE    'products'
))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 900);
```

ClickHouse connects to PostgreSQL using the standard libpq protocol and runs `SELECT * FROM products` on each refresh.

## Using a Custom Query

Filter or transform the data at the source:

```sql
SOURCE(POSTGRESQL(
    HOST     'pg.example.com'
    PORT     5432
    USER     'clickhouse_reader'
    PASSWORD 'secret'
    DB       'shop'
    QUERY    'SELECT product_id, sku, category, base_price FROM products WHERE is_active = true'
))
```

## SSL/TLS Connection

Enable SSL for encrypted connections to PostgreSQL:

```sql
SOURCE(POSTGRESQL(
    HOST     'pg.example.com'
    PORT     5432
    USER     'clickhouse_reader'
    PASSWORD 'secret'
    DB       'shop'
    TABLE    'products'
    SSL_MODE 'require'
))
```

## Reloading the Dictionary

```sql
SYSTEM RELOAD DICTIONARY product_dict;
```

## Querying with the Dictionary

```sql
SELECT
    order_id,
    product_id,
    dictGet('product_dict', 'category',   toUInt64(product_id)) AS category,
    dictGet('product_dict', 'base_price', toUInt64(product_id)) AS base_price,
    quantity * dictGet('product_dict', 'base_price', toUInt64(product_id)) AS line_total
FROM orders
WHERE event_date = today();
```

## Checking Dictionary Status

```sql
SELECT name, element_count, bytes_allocated, status, last_successful_update_time
FROM system.dictionaries
WHERE name = 'product_dict';
```

## Schema Qualification

If the table is in a non-default schema, use the `SCHEMA` parameter:

```sql
SOURCE(POSTGRESQL(
    HOST   'pg.example.com'
    PORT   5432
    USER   'clickhouse_reader'
    PASSWORD 'secret'
    DB     'shop'
    SCHEMA 'catalog'
    TABLE  'products'
))
```

## Connection Failures and Fallback

If the PostgreSQL server is unreachable during a scheduled reload, ClickHouse retains the last successfully loaded copy. The `last_exception` column in `system.dictionaries` records the error.

## Summary

PostgreSQL dictionary sources integrate ClickHouse directly with your operational PostgreSQL database. By specifying the `POSTGRESQL` source with appropriate credentials and a `HASHED` layout, you can keep dimension dictionaries refreshed automatically and use them for zero-JOIN enrichment in analytics queries.
