# How to Use Parameterized Queries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Parameterized Query, Security, SQL, Query

Description: Learn how to use parameterized queries in ClickHouse to prevent SQL injection, improve query plan reuse, and write safer application code.

---

Parameterized queries separate SQL structure from user-supplied values, preventing SQL injection and enabling query plan caching. ClickHouse supports parameters through its HTTP interface and client libraries.

## Why Parameterized Queries Matter

Without parameterization, concatenating user input into SQL is dangerous:

```python
# Unsafe - never do this
query = f"SELECT * FROM events WHERE user_id = '{user_input}'"
```

A malicious value like `' OR 1=1 --` would return all rows.

## HTTP Interface Parameters

Using the ClickHouse HTTP interface, pass parameters via query string:

```bash
curl "http://localhost:8123/?query=SELECT+*+FROM+events+WHERE+user_id={user_id:String}&user_id=abc123"
```

Parameters are declared with `{name:Type}` syntax inside the query.

## Supported Parameter Types

```text
String, Int8, Int16, Int32, Int64, UInt8, UInt16, UInt32, UInt64,
Float32, Float64, Date, DateTime, UUID
```

## Python Client Example

Using `clickhouse-driver`:

```python
from clickhouse_driver import Client

client = Client('localhost')

result = client.execute(
    "SELECT event, count() FROM events WHERE user_id = %(user_id)s GROUP BY event",
    {'user_id': 'abc123'}
)
```

## Go Client Example

```go
conn, _ := clickhouse.Open(&clickhouse.Options{Addr: []string{"localhost:9000"}})

rows, _ := conn.Query(ctx,
    "SELECT event FROM events WHERE user_id = ? AND ts > ?",
    "abc123", time.Now().Add(-24*time.Hour),
)
```

## Named Parameters in SQL Queries

For the native protocol, use positional `?` or named `%(name)s` depending on the client library:

```sql
SELECT
    toStartOfHour(ts) AS hour,
    count() AS events
FROM clickstream
WHERE
    project_id = {pid:UInt64}
    AND ts BETWEEN {start:DateTime} AND {end:DateTime}
GROUP BY hour
ORDER BY hour;
```

## Benefits for Query Cache

Parameterized queries allow ClickHouse to reuse cached query plans since the query text stays constant. This reduces parse overhead for high-frequency queries.

## Summary

Parameterized queries are a security baseline and a performance tool in ClickHouse. Use `{name:Type}` syntax via HTTP or your client library's binding mechanism to keep user data separate from SQL structure.
