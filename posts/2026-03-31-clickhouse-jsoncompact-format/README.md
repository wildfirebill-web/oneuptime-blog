# How to Use JSONCompact Format in ClickHouse

Author: [oneuptime](https://github.com/oneuptime)

Tags: ClickHouse, JSON, Data Engineering, API

Description: Explore ClickHouse's JSONCompact and related compact JSON formats for bandwidth-efficient API responses, browser clients, and query result serialization.

## What Is JSONCompact?

`JSONCompact` is a ClickHouse JSON output format that serializes query results as arrays of values rather than objects with named keys. The column names appear once in a header, and each row is an array. This reduces payload size significantly when column names are long or the result set has many rows.

Standard `JSON` output:

```text
{"data":[{"user_id":1,"action":"login","ts":"2025-01-01 10:00:00"},{"user_id":2,"action":"purchase","ts":"2025-01-01 10:01:00"}]}
```

`JSONCompact` output:

```text
{"meta":[{"name":"user_id","type":"UInt32"},{"name":"action","type":"String"},{"name":"ts","type":"DateTime"}],"data":[[1,"login","2025-01-01 10:00:00"],[2,"purchase","2025-01-01 10:01:00"]],"rows":2,...}
```

## JSONCompact Variants

ClickHouse offers several compact JSON formats:

| Format | Description |
|--------|-------------|
| `JSONCompact` | Full response envelope with meta, data (arrays), statistics |
| `JSONCompactEachRow` | One array per row, no envelope, like JSONEachRow but array-based |
| `JSONCompactEachRowWithNames` | First row contains column names, then data rows as arrays |
| `JSONCompactEachRowWithNamesAndTypes` | First two rows are names and types, then data rows |
| `JSONCompactStrings` | Like JSONCompact but all values serialized as strings |
| `JSONCompactStringsEachRow` | Like JSONCompactEachRow but all values as strings |

## Using JSONCompact for API Responses

Query via the HTTP interface and get compact output:

```bash
curl 'http://localhost:8123/?query=SELECT+user_id,action,ts+FROM+events+LIMIT+5+FORMAT+JSONCompact'
```

Response:

```text
{
  "meta": [
    {"name":"user_id","type":"UInt32"},
    {"name":"action","type":"String"},
    {"name":"ts","type":"DateTime"}
  ],
  "data": [
    [1,"login","2025-01-01 10:00:00"],
    [2,"purchase","2025-01-01 10:01:00"]
  ],
  "rows": 2,
  "statistics": {"elapsed":0.001,"rows_read":2,"bytes_read":48}
}
```

## JSONCompactEachRow

Use `JSONCompactEachRow` when you want the streaming one-line-per-row behavior of `JSONEachRow` but with arrays instead of objects:

```sql
SELECT user_id, action, ts
FROM events
FORMAT JSONCompactEachRow;
```

Output (no envelope, no metadata):

```text
[1,"login","2025-01-01 10:00:00"]
[2,"purchase","2025-01-01 10:01:00"]
[3,"logout","2025-01-01 10:02:00"]
```

This is useful for streaming consumption where the client already knows the schema.

## JSONCompactEachRowWithNamesAndTypes

This variant is self-describing and the most useful for generic clients:

```sql
SELECT user_id, action, ts
FROM events
LIMIT 3
FORMAT JSONCompactEachRowWithNamesAndTypes;
```

Output:

```text
["user_id","action","ts"]
["UInt32","String","DateTime"]
[1,"login","2025-01-01 10:00:00"]
[2,"purchase","2025-01-01 10:01:00"]
[3,"logout","2025-01-01 10:02:00"]
```

The first line contains column names, the second contains types, and subsequent lines are data rows.

## Inserting Data with JSONCompactEachRow

You can also INSERT data using compact array format:

```sql
INSERT INTO events FORMAT JSONCompactEachRow
[1,"login","2025-01-01 10:00:00"]
[2,"purchase","2025-01-01 10:01:00"]
```

With column names included:

```sql
INSERT INTO events FORMAT JSONCompactEachRowWithNames
["user_id","action","ts"]
[1,"login","2025-01-01 10:00:00"]
[2,"purchase","2025-01-01 10:01:00"]
```

## Bandwidth Comparison

For a table with columns `event_id`, `session_id`, `user_agent`, `url`, `referrer`, `ip`, `ts` and 1000 rows:

| Format | Approximate Size |
|--------|-----------------|
| JSONEachRow | 450 KB |
| JSONCompactEachRow | 280 KB |
| JSONCompactEachRowWithNamesAndTypes | 282 KB |
| Parquet | 85 KB |

Compact formats save roughly 30-40% compared to object-per-row formats while remaining human-readable.

## Using JSONCompactStrings

When your client code handles all values as strings (common in JavaScript frontends), `JSONCompactStrings` avoids type coercion surprises:

```bash
curl 'http://localhost:8123/?query=SELECT+user_id,ts+FROM+events+LIMIT+2+FORMAT+JSONCompactStrings'
```

Output:

```text
{"meta":[...],"data":[["1","2025-01-01 10:00:00"],["2","2025-01-01 10:01:00"]],...}
```

All values including integers and floats become JSON strings.

## Practical Tips

1. Use `JSONCompact` for dashboards and API endpoints where you control the client code.
2. Use `JSONCompactEachRowWithNamesAndTypes` when building generic clients that auto-detect schema.
3. Use `JSONCompactEachRow` for streaming pipelines where the schema is known ahead of time.
4. For very large result sets, switch to `Arrow` or `Parquet` for better performance.

## Conclusion

JSONCompact and its variants offer a useful middle ground between human-readability and bandwidth efficiency. They are particularly well-suited for web APIs, dashboard backends, and any scenario where JSON is preferred but payload size matters.

**Related Reading:**

- [How to Use JSONEachRow Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-jsoneachrow-format/view)
- [How to Export ClickHouse Data to Different File Formats](https://oneuptime.com/blog/post/2026-03-31-clickhouse-export-file-formats/view)
- [How to Use TabSeparated Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-tabseparated-format/view)
