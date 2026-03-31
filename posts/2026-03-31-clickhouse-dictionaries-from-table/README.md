# How to Load Dictionaries from ClickHouse Tables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, Table Source, Data Loading, In-Memory

Description: Learn how to load ClickHouse dictionaries from a ClickHouse table, enabling fast in-memory lookups backed by your own MergeTree or ReplacingMergeTree tables.

---

Using a ClickHouse table as the dictionary source is the most common and performant option when your dimension data already lives in ClickHouse. The dictionary loads the entire table (or a custom query result) into memory and serves lookups without any JOIN at query time.

## Why Use a ClickHouse Table as Source

- No external dependency - everything inside ClickHouse
- Supports all table engines including `MergeTree`, `ReplacingMergeTree`, and `Dictionary`
- Custom SQL queries let you filter, join, or transform before loading
- Updates in the source table are picked up at the next `LIFETIME` cycle

## Creating the Source Table

```sql
CREATE TABLE currency_rates
(
    currency_code String,
    rate_to_usd   Float64,
    updated_at    DateTime
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY currency_code;

INSERT INTO currency_rates VALUES
('EUR', 1.08, now()),
('GBP', 1.27, now()),
('JPY', 0.0067, now());
```

## Creating the Dictionary

```sql
CREATE DICTIONARY currency_dict
(
    currency_code String,
    rate_to_usd   Float64
)
PRIMARY KEY currency_code
SOURCE(CLICKHOUSE(
    HOST 'localhost' PORT 9000
    USER 'default' PASSWORD ''
    DB   'default'
    TABLE 'currency_rates'
))
LAYOUT(HASHED())
LIFETIME(MIN 60 MAX 120);
```

For distributed setups, use the actual server hostname and credentials.

## Using a Custom Query as Source

```sql
SOURCE(CLICKHOUSE(
    HOST  'localhost' PORT 9000
    USER  'default' PASSWORD ''
    DB    'default'
    QUERY 'SELECT currency_code, argMax(rate_to_usd, updated_at) FROM currency_rates GROUP BY currency_code'
))
```

This ensures only the latest rate per currency is loaded, even if the underlying table has duplicates.

## Reloading the Dictionary

```sql
SYSTEM RELOAD DICTIONARY currency_dict;
```

## Querying with the Dictionary

```sql
SELECT
    order_id,
    amount_local,
    currency_code,
    amount_local * dictGet('currency_dict', 'rate_to_usd', currency_code) AS amount_usd
FROM orders
WHERE event_date = today();
```

## Using dictGetOrDefault

```sql
SELECT
    order_id,
    amount_local * dictGetOrDefault('currency_dict', 'rate_to_usd', currency_code, 1.0) AS amount_usd
FROM orders;
```

Returns `1.0` if the currency code is not found in the dictionary.

## Checking Dictionary Metadata

```sql
SELECT name, element_count, bytes_allocated, last_successful_update_time, status
FROM system.dictionaries
WHERE name = 'currency_dict';
```

## Layout Choices

| Layout | Best For |
|--------|----------|
| `FLAT` | Integer keys up to a few million rows |
| `HASHED` | String or large integer keys |
| `SPARSE_HASHED` | Large dictionaries with memory constraints |
| `CACHE` | Very large dictionaries where partial caching is acceptable |

## Summary

Loading dictionaries from ClickHouse tables is the simplest and most efficient approach when your dimension data is already in ClickHouse. By pairing a `CLICKHOUSE` source with a `HASHED` layout and configuring an appropriate `LIFETIME`, you get fast in-memory lookups that automatically stay in sync with the backing table.
