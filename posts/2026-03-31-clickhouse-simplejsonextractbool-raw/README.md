# How to Use simpleJSONExtractBool() and simpleJSONExtractRaw() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, JSON, simpleJSONExtractBool, simpleJSONExtractRaw, Function

Description: Learn how to use simpleJSONExtractBool() and simpleJSONExtractRaw() in ClickHouse for fast boolean and raw value extraction from flat JSON strings.

---

As part of the `simpleJSON*` function family, `simpleJSONExtractBool()` and `simpleJSONExtractRaw()` provide lightweight extraction of boolean and raw JSON values from flat JSON objects. They trade full JSON compliance for speed, making them excellent for high-throughput log processing scenarios.

## How These Functions Work

- `simpleJSONExtractBool(json, field)` - returns `1` if the field is the literal `true`, `0` otherwise (including for missing fields and non-boolean values).
- `simpleJSONExtractRaw(json, field)` - returns the raw value string at the given key without any type conversion. Numbers, strings, booleans, and objects are all returned as their literal JSON representation (including quotes for strings).

Both functions only support flat top-level key lookups.

## Syntax

```sql
simpleJSONExtractBool(json, field_name)
simpleJSONExtractRaw(json, field_name)
```

## Examples

### Extracting Boolean Flags

```sql
SELECT
    simpleJSONExtractBool('{"enabled": true,  "debug": false}', 'enabled') AS enabled,
    simpleJSONExtractBool('{"enabled": true,  "debug": false}', 'debug')   AS debug_mode;
```

```text
enabled  debug_mode
1        0
```

### Missing Key Returns 0 for Bool

```sql
SELECT simpleJSONExtractBool('{"a": true}', 'b') AS missing;
```

```text
missing
0
```

### Extracting Raw Values

`simpleJSONExtractRaw()` preserves the original JSON value representation:

```sql
SELECT
    simpleJSONExtractRaw('{"name": "Alice", "age": 30, "active": true}', 'name')   AS raw_str,
    simpleJSONExtractRaw('{"name": "Alice", "age": 30, "active": true}', 'age')    AS raw_int,
    simpleJSONExtractRaw('{"name": "Alice", "age": 30, "active": true}', 'active') AS raw_bool;
```

```text
raw_str    raw_int  raw_bool
"Alice"    30       true
```

### Using Raw for Type-Flexible Columns

`simpleJSONExtractRaw()` is useful when a field might contain different types:

```sql
SELECT
    simpleJSONExtractRaw('{"value": 42}',      'value') AS numeric_val,
    simpleJSONExtractRaw('{"value": "hello"}', 'value') AS string_val,
    simpleJSONExtractRaw('{"value": [1,2,3]}', 'value') AS array_val;
```

```text
numeric_val  string_val  array_val
42           "hello"     [1,2,3]
```

### Complete Working Example

Process feature toggle events and analyze activation rates:

```sql
CREATE TABLE feature_events
(
    event_id  UInt64,
    log_json  String,
    logged_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY event_id;

INSERT INTO feature_events (event_id, log_json) VALUES
    (1, '{"feature": "dark_mode",     "enabled": true,  "user": "u1"}'),
    (2, '{"feature": "dark_mode",     "enabled": false, "user": "u2"}'),
    (3, '{"feature": "beta_search",   "enabled": true,  "user": "u1"}'),
    (4, '{"feature": "beta_search",   "enabled": true,  "user": "u3"}'),
    (5, '{"feature": "dark_mode",     "enabled": true,  "user": "u3"}');

SELECT
    simpleJSONExtractRaw(log_json, 'feature')      AS feature,
    countIf(simpleJSONExtractBool(log_json, 'enabled') = 1) AS enabled_count,
    countIf(simpleJSONExtractBool(log_json, 'enabled') = 0) AS disabled_count
FROM feature_events
GROUP BY feature
ORDER BY feature;
```

```text
feature        enabled_count  disabled_count
"beta_search"  2              0
"dark_mode"    2              1
```

## Summary

`simpleJSONExtractBool()` returns `1` or `0` for boolean fields in flat JSON objects, and `simpleJSONExtractRaw()` returns the raw JSON value of any field as a string without type conversion. Both functions are part of the high-performance `simpleJSON*` family best suited for flat JSON log processing. Note that `simpleJSONExtractRaw()` includes quotes around string values - use `simpleJSONExtractString()` if you need the string without quotes.
