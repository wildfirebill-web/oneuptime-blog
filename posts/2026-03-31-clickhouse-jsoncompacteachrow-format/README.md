# How to Use JSONCompactEachRow Format in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSONCompactEachRow Format, JSON, Data Export, Compact Format

Description: Learn how to use JSONCompactEachRow and its variants in ClickHouse for space-efficient JSON output where each row is a JSON array instead of an object.

---

JSONCompactEachRow is a space-efficient variant of JSONEachRow where each row is serialized as a JSON array instead of a JSON object. By omitting field names from every row, it significantly reduces output size - especially useful for large exports or network-constrained environments.

## JSONCompactEachRow vs JSONEachRow

JSONEachRow (standard):
```text
{"id":1,"name":"Alice","value":3.14}
{"id":2,"name":"Bob","value":2.71}
```

JSONCompactEachRow (compact, arrays):
```text
[1,"Alice",3.14]
[2,"Bob",2.71]
```

The compact format can reduce output size by 30-60% for wide tables.

## Basic Usage

```sql
SELECT id, name, value
FROM my_table
LIMIT 5
FORMAT JSONCompactEachRow;
```

## JSONCompactEachRowWithNames

Adds a header row with column names:

```text
["id","name","value"]
[1,"Alice",3.14]
[2,"Bob",2.71]
```

```sql
SELECT id, name, value
FROM my_table
FORMAT JSONCompactEachRowWithNames;
```

## JSONCompactEachRowWithNamesAndTypes

Adds both column names and types as separate header rows:

```text
["id","name","value"]
["UInt64","String","Float64"]
[1,"Alice",3.14]
[2,"Bob",2.71]
```

```sql
SELECT id, name, value
FROM my_table
FORMAT JSONCompactEachRowWithNamesAndTypes;
```

This is useful when the consumer needs type information without a separate schema registry.

## Importing Data

JSONCompactEachRow works as both input and output. You can import array-encoded JSON:

```bash
clickhouse-client \
    --query "INSERT INTO my_table FORMAT JSONCompactEachRow" \
    << 'EOF'
[1,"Alice",3.14]
[2,"Bob",2.71]
EOF
```

With the WithNames variant, the header row is skipped during import:

```bash
clickhouse-client \
    --query "INSERT INTO my_table FORMAT JSONCompactEachRowWithNames" \
    < compact_data_with_header.json
```

## Use Cases

JSONCompactEachRow is well-suited for:
- High-volume data exports where file size matters
- Internal microservice APIs where consumers know the schema
- Bulk transfers between ClickHouse instances
- Reducing Kafka message sizes in JSON-serialized pipelines

## Estimating Size Savings

For a table with columns `(event_id UInt64, user_id UInt32, event_type String, ts DateTime)`:

```sql
-- Compare sizes
SELECT
    'JSONEachRow' AS format,
    length(formatRow('JSONEachRow', 12345, 42, 'page_view', now())) AS bytes
UNION ALL
SELECT
    'JSONCompactEachRow',
    length(formatRow('JSONCompactEachRow', 12345, 42, 'page_view', now()))
```

## Summary

JSONCompactEachRow and its WithNames/WithNamesAndTypes variants provide a compact, array-encoded alternative to standard JSONEachRow. They are particularly valuable for high-volume exports, internal APIs, and data pipelines where reducing payload size improves throughput. Use the WithNamesAndTypes variant when consumers need schema information embedded in the file, and the plain variant for maximum compactness when the schema is known ahead of time.
