# How to Use JSONStringsEachRow Format in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSONStringsEachRow Format, JSON Format, Data Export, Type Safety

Description: Learn how JSONStringsEachRow format in ClickHouse serializes all values as strings, avoiding type coercion issues when exporting data to weakly-typed systems.

---

`JSONStringsEachRow` is a variant of the `JSONEachRow` format in ClickHouse where all column values are encoded as JSON strings, regardless of their actual data type. Numbers, dates, booleans - everything becomes a string. This is useful when the consuming system does not handle numeric types reliably or when you need consistent string representations across all columns.

## How JSONStringsEachRow Differs from JSONEachRow

With standard `JSONEachRow`, numeric values are emitted as JSON numbers:

```sql
SELECT user_id, score, event_date
FROM events
LIMIT 2
FORMAT JSONEachRow;
```

```json
{"user_id":1001,"score":9.5,"event_date":"2026-03-31"}
{"user_id":1002,"score":7.2,"event_date":"2026-03-31"}
```

With `JSONStringsEachRow`, all values including numbers are quoted strings:

```sql
SELECT user_id, score, event_date
FROM events
LIMIT 2
FORMAT JSONStringsEachRow;
```

```json
{"user_id":"1001","score":"9.5","event_date":"2026-03-31"}
{"user_id":"1002","score":"7.2","event_date":"2026-03-31"}
```

## When to Use JSONStringsEachRow

### Compatibility with Weakly-Typed Systems

Some downstream systems (legacy APIs, JavaScript environments, certain message queue consumers) cannot distinguish between numeric and string JSON values reliably. Emitting all values as strings prevents type coercion bugs:

```sql
-- Export events for a legacy JavaScript API that expects all strings
SELECT
    event_id,
    user_id,
    event_name,
    toUnixTimestamp(event_date) AS timestamp
FROM events
WHERE event_date = today()
FORMAT JSONStringsEachRow;
```

### Preserving Large Integer Precision

JavaScript's `JSON.parse()` silently loses precision for integers larger than 2^53. By emitting as strings, you preserve full UInt64 precision:

```sql
SELECT
    high_cardinality_id,  -- UInt64 up to 18 digits
    event_name
FROM events
LIMIT 5
FORMAT JSONStringsEachRow;
```

```json
{"high_cardinality_id":"12345678901234567","event_name":"purchase"}
```

## Ingesting JSONStringsEachRow Data

ClickHouse can also read `JSONStringsEachRow` format on INSERT. Values in string form are automatically cast to the target column type:

```bash
echo '{"user_id":"101","score":"9.5","event_date":"2026-03-31"}' | \
  clickhouse-client --query="INSERT INTO events FORMAT JSONStringsEachRow"
```

ClickHouse will cast `"101"` to UInt64 and `"9.5"` to Float64 automatically.

## Related Variants

ClickHouse has a family of JSONStrings formats:

```sql
-- JSONStrings: full JSON object with metadata, all values as strings
SELECT * FROM events LIMIT 3 FORMAT JSONStrings;

-- JSONStringsEachRow: NDJSON with all values as strings
SELECT * FROM events LIMIT 3 FORMAT JSONStringsEachRow;

-- JSONCompactStringsEachRow: compact array format, all values as strings
SELECT * FROM events LIMIT 3 FORMAT JSONCompactStringsEachRow;
```

## Comparing JSON String Formats

| Format | Structure | Values |
|---|---|---|
| JSONEachRow | One object per line | Native types |
| JSONStringsEachRow | One object per line | All strings |
| JSONCompactEachRow | One array per line | Native types |
| JSONCompactStringsEachRow | One array per line | All strings |

## Practical Example: Kafka Export

When exporting data to Kafka consumers that may be written in JavaScript or other dynamically typed languages:

```sql
SELECT
    event_id,
    user_id,
    event_name,
    formatDateTime(event_date, '%Y-%m-%d') AS date
FROM events
WHERE event_date = today()
FORMAT JSONStringsEachRow
SETTINGS output_format_json_quote_64bit_integers = 1;
```

## Summary

`JSONStringsEachRow` is the right choice when downstream consumers expect all JSON values as strings, or when large integer precision must be preserved. It is a drop-in replacement for `JSONEachRow` with the single difference that every value is quoted. Use it for compatibility with JavaScript clients, legacy APIs, or any system that cannot handle mixed JSON types reliably.
