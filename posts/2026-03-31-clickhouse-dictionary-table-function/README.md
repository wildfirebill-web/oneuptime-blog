# How to Use dictionary() Table Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, dictionary(), Table Function, Dictionary, SQL, Lookup, Analytics

Description: Learn how to use the dictionary() table function in ClickHouse to query dictionary data as a table, enabling SQL-style exploration of dictionary contents.

---

The `dictionary(dict_name)` table function exposes a ClickHouse dictionary as a queryable table. While dictionaries are typically accessed via `dictGet*` functions for point lookups, `dictionary()` lets you SELECT, filter, aggregate, and JOIN dictionary data using standard SQL syntax - useful for exploration, validation, and bulk lookups.

## Prerequisites

A dictionary must exist in ClickHouse before using `dictionary()`. Create one from a ClickHouse source:

```sql
CREATE DICTIONARY country_lookup (
    country_code String,
    country_name String,
    region       String,
    is_eu        UInt8
)
PRIMARY KEY country_code
SOURCE(CLICKHOUSE(TABLE 'country_dim'))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 600);
```

## Basic SELECT from a Dictionary

```sql
SELECT *
FROM dictionary('country_lookup')
LIMIT 10;
```

```text
country_code  country_name    region  is_eu
US            United States   AMER    0
DE            Germany         EMEA    1
JP            Japan           APAC    0
GB            United Kingdom  EMEA    1
```

## Filtering Dictionary Data

```sql
-- Find all EU countries
SELECT country_code, country_name
FROM dictionary('country_lookup')
WHERE is_eu = 1
ORDER BY country_name;
```

## Counting and Aggregating

```sql
SELECT
    region,
    count()   AS country_count,
    sum(is_eu) AS eu_country_count
FROM dictionary('country_lookup')
GROUP BY region
ORDER BY country_count DESC;
```

## Comparing dictGet vs dictionary() Lookups

For point lookups on known keys, `dictGet` is faster:

```sql
-- Fast single-key lookup
SELECT dictGet('country_lookup', 'country_name', 'DE') AS name;

-- dictionary() is better for bulk inspection or unknown-key scenarios
SELECT *
FROM dictionary('country_lookup')
WHERE country_code IN ('DE', 'US', 'JP');
```

## Joining a Dictionary with a Fact Table

```sql
SELECT
    e.event_type,
    c.region,
    count() AS event_count
FROM events AS e
LEFT JOIN dictionary('country_lookup') AS c ON e.country_code = c.country_code
WHERE e.event_time >= today() - 7
GROUP BY e.event_type, c.region
ORDER BY event_count DESC;
```

This differs from `dictGet` JOIN-style lookups in that it materializes all dictionary rows and performs a standard hash join.

## Inspecting Dictionary Metadata

```sql
-- Count rows in a dictionary
SELECT count() FROM dictionary('country_lookup');

-- Inspect schema
DESCRIBE TABLE dictionary('country_lookup');
```

## Checking Dictionary Load Status

For validation, compare `dictionary()` row count against the source:

```sql
SELECT
    (SELECT count() FROM dictionary('country_lookup')) AS dict_count,
    (SELECT count() FROM country_dim) AS source_count;
```

## Reloading and Verifying

After a `SYSTEM RELOAD DICTIONARY country_lookup`, re-query `dictionary()` to confirm the new data is present:

```sql
SYSTEM RELOAD DICTIONARY country_lookup;

SELECT count() FROM dictionary('country_lookup');
```

## Summary

The `dictionary()` table function exposes any loaded ClickHouse dictionary as a standard SQL table. Use it for exploration, validation, bulk key lookups, and standard SQL joins against dictionary data. For high-throughput single-key lookups in fact table queries, `dictGet` functions remain faster because they bypass the SQL planner.
