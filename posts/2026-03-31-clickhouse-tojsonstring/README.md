# How to Use toJSONString() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Analytics, Query

Description: Learn how toJSONString() serializes ClickHouse row values, arrays, and tuples into JSON strings, enabling JSON output formatting and downstream API integration.

---

`toJSONString` serializes a ClickHouse value into its JSON string representation. It works on scalars, arrays, tuples, maps, and `Nullable` types, and it respects ClickHouse's type system when choosing JSON encoding. This makes it useful for building JSON payloads for API responses, serializing complex types before writing to an external system, and debugging what a value looks like when rendered as JSON.

## Basic Usage with Scalars

```sql
-- Serialize different scalar types to JSON
SELECT
    toJSONString(42)          AS int_json,
    toJSONString(3.14)        AS float_json,
    toJSONString('hello')     AS string_json,
    toJSONString(true)        AS bool_json,
    toJSONString(NULL)        AS null_json;
```

```text
int_json  float_json  string_json  bool_json  null_json
42        3.14        "hello"      true       null
```

## Serializing Arrays

```sql
-- Convert a native array to a JSON array string
SELECT toJSONString([1, 2, 3]) AS json_array;
```

```text
json_array
[1,2,3]
```

## Serializing Tuples

```sql
-- Tuples serialize as JSON arrays
SELECT toJSONString(tuple(1, 'Alice', true)) AS json_tuple;
```

```text
json_tuple
[1,"Alice",true]
```

## Serializing a Map

```sql
-- Maps serialize as JSON objects
SELECT toJSONString(map('env', 'prod', 'region', 'us-east')) AS json_map;
```

```text
json_map
{"env":"prod","region":"us-east"}
```

## Building a JSON Object from Columns

Combine `toJSONString` with string concatenation to build a JSON object from individual columns.

```sql
-- Construct a JSON object row from table columns
SELECT concat(
    '{"user_id":',    toJSONString(user_id),
    ',"email":',      toJSONString(email),
    ',"created_at":', toJSONString(toString(created_at)),
    '}'
) AS user_json
FROM users
LIMIT 5;
```

## Serializing Nullable Values

```sql
-- NULL values appear as JSON null
SELECT
    toJSONString(toNullable(42))   AS some_value,
    toJSONString(toNullable(NULL)) AS null_value;
```

```text
some_value  null_value
42          null
```

## Using toJSONString in an INSERT Pipeline

```sql
-- Write serialized arrays into a JSON column
INSERT INTO export_queue (row_id, json_payload)
SELECT
    event_id,
    toJSONString(array(
        JSONExtractString(payload, 'user_id'),
        JSONExtractString(payload, 'event_type'),
        toString(event_time)
    ))
FROM events
WHERE toDate(event_time) = yesterday();
```

## Exporting Aggregated Arrays as JSON

```sql
-- Return a JSON array of the top 5 event types per day
SELECT
    toDate(event_time)                              AS event_date,
    toJSONString(
        arraySlice(
            groupArray(event_type),
            1, 5
        )
    )                                               AS top_event_types_json
FROM events
GROUP BY event_date
ORDER BY event_date DESC
LIMIT 10;
```

## Summary

`toJSONString` converts ClickHouse values of any supported type into their JSON string representation. Use it to serialize arrays and maps for downstream consumers, build JSON payloads inline from columnar data, and serialize `Nullable` values that require explicit `null` output. For constructing full JSON objects, combine it with `concat` or use the `FORMAT JSON` output format when running queries that need structured JSON output.
