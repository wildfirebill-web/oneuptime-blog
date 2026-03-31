# How to Use Integrated Search and JSON in Redis 8.0+

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Search, JSON, Database, Full-Text Search

Description: Learn how to use Redis 8.0's built-in search and JSON capabilities to query, index, and filter structured data without external modules.

---

Redis 8.0 ships with integrated support for JSON storage and full-text search as first-class features, previously available only through the RedisSearch and RedisJSON modules. This guide shows you how to get started with both.

## Storing JSON Documents

Use `JSON.SET` to store structured documents:

```bash
JSON.SET user:1001 $ '{"name":"Alice","age":30,"role":"engineer","city":"Berlin"}'
JSON.SET user:1002 $ '{"name":"Bob","age":25,"role":"designer","city":"London"}'
JSON.SET user:1003 $ '{"name":"Carol","age":35,"role":"engineer","city":"Berlin"}'
```

Retrieve specific fields using JSONPath:

```bash
JSON.GET user:1001 $.name
# "Alice"

JSON.MGET user:1001 user:1002 $.city
# ["Berlin", "London"]
```

## Creating a Search Index

Before querying, create an index that maps JSON fields to searchable attributes:

```bash
FT.CREATE idx:users
  ON JSON
  PREFIX 1 user:
  SCHEMA
    $.name AS name TEXT
    $.age  AS age  NUMERIC SORTABLE
    $.role AS role TAG
    $.city AS city TAG
```

This index tells Redis to watch all keys starting with `user:` and index the four fields.

## Running Search Queries

Full-text search on the `name` field:

```bash
FT.SEARCH idx:users "@name:Alice"
```

Filter by tag and numeric range:

```bash
FT.SEARCH idx:users "@role:{engineer} @age:[28 40]"
```

Sort results by age ascending:

```bash
FT.SEARCH idx:users "@city:{Berlin}" SORTBY age ASC
```

## Aggregation Pipelines

Redis search supports aggregations similar to SQL GROUP BY:

```bash
FT.AGGREGATE idx:users "*"
  GROUPBY 1 @city
  REDUCE COUNT 0 AS total
  SORTBY 2 @total DESC
```

This counts users per city and orders the result by count.

## Updating Indexed Documents

Updating a JSON document automatically re-indexes it:

```bash
JSON.SET user:1002 $.city '"Berlin"'
```

The index reflects the change immediately on the next query.

## Using Python

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Store document
r.execute_command("JSON.SET", "user:2001", "$",
    json.dumps({"name": "Dana", "age": 28, "role": "ops", "city": "Paris"}))

# Search
results = r.execute_command(
    "FT.SEARCH", "idx:users", "@city:{Paris}", "RETURN", "1", "$.name"
)
print(results)
```

## Index Management Tips

- Use `FT.INFO idx:users` to inspect index stats and field counts.
- Drop and recreate an index with `FT.DROPINDEX idx:users` when schema changes.
- Prefix multiple document types in one index by adding more `PREFIX` entries.
- For large datasets, build indexes during low-traffic windows to avoid latency spikes.

## Summary

Redis 8.0 brings search and JSON natively into the core, removing the need to run separate module builds. You can store structured documents, define field-level indexes, and run full-text, tag, and numeric queries with a simple command set. Aggregations make it practical for analytics without a separate query engine.
