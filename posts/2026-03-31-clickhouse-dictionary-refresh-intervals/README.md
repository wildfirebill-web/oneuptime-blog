# How to Configure Dictionary Refresh Intervals in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, Refresh, Lifetime, Configuration

Description: Learn how to configure ClickHouse dictionary refresh intervals using LIFETIME settings to balance data freshness with load on the source system.

---

ClickHouse dictionaries are loaded into memory and refreshed periodically. The `LIFETIME` parameter controls how often ClickHouse reloads the dictionary from its source. Choosing the right interval balances data freshness against the load you put on your source database or endpoint.

## The LIFETIME Parameter

`LIFETIME` takes a minimum and maximum value in seconds:

```sql
LIFETIME(MIN 300 MAX 600);
```

ClickHouse schedules the next reload at a random time between `MIN` and `MAX` seconds after the last successful load. The randomness spreads reloads across the interval to avoid thundering herd on the source.

## Static Dictionary - Never Reload

Set both `MIN` and `MAX` to zero to load once at startup and never reload:

```sql
CREATE DICTIONARY static_country_dict
(
    country_code String,
    country_name String
)
PRIMARY KEY country_code
SOURCE(FILE(PATH '/var/lib/clickhouse/user_files/countries.csv' FORMAT 'CSVWithNames'))
LAYOUT(HASHED())
LIFETIME(MIN 0 MAX 0);
```

## Frequent Refresh for Near-Real-Time Data

For data that changes often (currency rates, feature flags):

```sql
LIFETIME(MIN 60 MAX 120);
```

## Infrequent Refresh for Slowly Changing Data

For data that rarely changes (country codes, status enumerations):

```sql
LIFETIME(MIN 86400 MAX 172800);
```

## Checking When a Dictionary Was Last Updated

```sql
SELECT
    name,
    last_successful_update_time,
    loading_duration,
    element_count
FROM system.dictionaries
WHERE name = 'currency_dict';
```

## Forcing an Immediate Reload

Regardless of the scheduled interval:

```sql
SYSTEM RELOAD DICTIONARY currency_dict;
```

Reload all dictionaries at once:

```sql
SYSTEM RELOAD DICTIONARIES;
```

## Invalidation-Based Reloads

For ClickHouse table sources, you can use a modification check to avoid loading data that has not changed. Add a `INVALIDATE_QUERY` to skip the reload when the result of the query is unchanged:

```sql
SOURCE(CLICKHOUSE(
    HOST  'localhost' PORT 9000
    USER  'default' PASSWORD ''
    DB    'default'
    TABLE 'currency_rates'
    INVALIDATE_QUERY 'SELECT max(updated_at) FROM currency_rates'
))
LIFETIME(MIN 60 MAX 120);
```

ClickHouse runs the `INVALIDATE_QUERY` before each scheduled reload. If the result matches the previous run, the reload is skipped.

## MySQL and PostgreSQL Source Invalidation

```sql
SOURCE(MYSQL(
    HOST     'mysql.example.com'
    PORT     3306
    USER     'reader' PASSWORD 'secret'
    DB       'app_db'
    TABLE    'products'
    INVALIDATE_QUERY 'SELECT MAX(updated_at) FROM products'
))
LIFETIME(MIN 120 MAX 240);
```

## Background Loading Behavior

When a reload is triggered, ClickHouse loads the new version in the background while still serving queries from the old version. The switch happens atomically once the new load completes.

## Summary

The `LIFETIME` parameter is the primary control for dictionary freshness in ClickHouse. Use `MIN 0 MAX 0` for static data, short intervals for frequently changing data, and `INVALIDATE_QUERY` to avoid unnecessary reloads when the source data has not changed. Background loading ensures queries are never blocked during a dictionary refresh.
