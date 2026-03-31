# How to Use JSONExtractKeysAndValues() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Json, Analytics, Query

Description: Learn how JSONExtractKeysAndValues() returns all key-value pairs from a JSON object as an Array of tuples in ClickHouse, enabling dynamic key iteration and pivot queries.

---

`JSONExtractKeysAndValues` extracts all key-value pairs from a JSON object and returns them as an `Array(Tuple(String, T))` where `T` is the type you specify for the values. This is useful when you do not know the keys ahead of time and want to iterate over every field in an object, build pivot tables, or search across all values without naming each key explicitly.

## Basic Usage

The second argument is the value type. Use `String` for mixed-type objects because ClickHouse will cast each value.

```sql
-- Extract every key-value pair as (String, String) tuples
SELECT JSONExtractKeysAndValues('{"env": "prod", "region": "us-east-1", "version": "2.1"}', String);
```

```text
[('env','prod'),('region','us-east-1'),('version','2.1')]
```

## Extracting Numeric Key-Value Pairs

```sql
-- Extract metric names and their Int64 values from a JSON object
SELECT JSONExtractKeysAndValues('{"cpu": 72, "mem": 4096, "disk": 120}', Int64);
```

```text
[('cpu',72),('mem',4096),('disk',120)]
```

## Iterating Over All Keys in a Table Column

```sql
-- Flatten all properties into key/value rows for a given event
SELECT
    event_id,
    kv.1 AS property_key,
    kv.2 AS property_value
FROM events
ARRAY JOIN JSONExtractKeysAndValues(properties_json, String) AS kv
LIMIT 20;
```

## Searching Across All Values

```sql
-- Find events where any property value contains the word 'error'
SELECT
    event_id,
    event_time
FROM (
    SELECT
        event_id,
        event_time,
        JSONExtractKeysAndValues(properties_json, String) AS kv_pairs
    FROM events
)
WHERE arrayExists(kv -> positionCaseInsensitive(kv.2, 'error') > 0, kv_pairs)
ORDER BY event_time DESC
LIMIT 10;
```

## Building a Pivot from Dynamic Keys

```sql
-- Aggregate values per key across all rows (dynamic pivot skeleton)
SELECT
    kv.1   AS metric_name,
    avg(toFloat64(kv.2)) AS avg_value,
    count() AS readings
FROM metrics
ARRAY JOIN JSONExtractKeysAndValues(dimensions, String) AS kv
GROUP BY kv.1
ORDER BY avg_value DESC;
```

## Navigating to a Nested Object First

Pass additional path arguments before the type to navigate into a nested object.

```sql
-- Extract all key-value pairs from the nested "labels" object
SELECT
    resource_id,
    kv.1 AS label_key,
    kv.2 AS label_value
FROM resources
ARRAY JOIN JSONExtractKeysAndValues(resource_json, 'labels', String) AS kv;
```

## Counting How Many Keys an Object Has

```sql
-- Objects with more than 5 properties are considered wide events
SELECT
    event_id,
    length(JSONExtractKeysAndValues(properties_json, String)) AS key_count
FROM events
WHERE length(JSONExtractKeysAndValues(properties_json, String)) > 5
ORDER BY key_count DESC
LIMIT 20;
```

## Summary

`JSONExtractKeysAndValues` provides a way to iterate over all fields of a JSON object without enumerating key names ahead of time. Pair it with `ARRAY JOIN` to flatten key-value pairs into rows, `arrayExists` to search across all values, and nested path arguments to target a sub-object. Specify the tightest value type that fits your data to avoid unnecessary string casting.
