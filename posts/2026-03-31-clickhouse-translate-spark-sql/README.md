# How to Translate Spark SQL to ClickHouse SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Spark, Migration, Query Translation, Analytics

Description: A guide to converting Spark SQL queries to ClickHouse SQL, with function mapping tables, array handling differences, and practical query examples.

---

## Spark SQL vs ClickHouse

Spark SQL runs on distributed Spark clusters and suits large batch transformations. ClickHouse excels at low-latency analytical queries on pre-loaded columnar data. Teams often run Spark for ETL and ClickHouse for serving the analytical layer.

## Data Type Mapping

```sql
-- Spark SQL            ClickHouse
INT                  -> Int32
LONG / BIGINT        -> Int64
DOUBLE               -> Float64
DECIMAL(p,s)         -> Decimal(p,s)
STRING               -> String
BOOLEAN              -> UInt8
TIMESTAMP            -> DateTime64(6)
DATE                 -> Date
ARRAY<T>             -> Array(T)
MAP<K,V>             -> Map(K,V)
STRUCT<...>          -> Tuple(...)
```

## Date Truncation

```sql
-- Spark SQL
SELECT date_trunc('MONTH', event_time) AS month FROM events;

-- ClickHouse
SELECT toStartOfMonth(event_time) AS month FROM events;
```

Spark's `date_trunc` granularity strings map to ClickHouse functions:

```text
YEAR   -> toStartOfYear
MONTH  -> toStartOfMonth
WEEK   -> toStartOfWeek
DAY    -> toStartOfDay
HOUR   -> toStartOfHour
MINUTE -> toStartOfMinute
```

## DATEDIFF

```sql
-- Spark SQL (returns days)
SELECT datediff(end_date, start_date) AS days FROM tasks;

-- ClickHouse
SELECT dateDiff('day', start_date, end_date) AS days FROM tasks;
```

Note: argument order is reversed in ClickHouse compared to Spark.

## Explode vs arrayJoin

```sql
-- Spark SQL
SELECT user_id, explode(tags) AS tag FROM users;

-- ClickHouse
SELECT user_id, arrayJoin(tags) AS tag FROM users;
```

## collect_list and collect_set

```sql
-- Spark SQL
SELECT user_id, collect_list(product_id) AS all_products,
       collect_set(category) AS unique_categories
FROM orders GROUP BY user_id;

-- ClickHouse
SELECT user_id, groupArray(product_id) AS all_products,
       groupUniqArray(category) AS unique_categories
FROM orders GROUP BY user_id;
```

## nvl and coalesce

```sql
-- Spark SQL
SELECT nvl(revenue, 0.0) FROM transactions;

-- ClickHouse
SELECT ifNull(revenue, 0.0) FROM transactions;
```

## String Formatting

```sql
-- Spark SQL
SELECT format_string('user_%d', user_id) FROM users;

-- ClickHouse
SELECT format('user_{}', user_id) FROM users;
-- or
SELECT concat('user_', toString(user_id)) FROM users;
```

## Summary

Spark SQL to ClickHouse translation primarily involves replacing `date_trunc` with `toStartOf*` helpers, swapping `explode` for `arrayJoin`, converting `collect_list`/`collect_set` to `groupArray`/`groupUniqArray`, and noting the reversed argument order in `dateDiff`. Type names shift from Spark's Hive-style names to ClickHouse's explicit signed integer and Decimal types.
