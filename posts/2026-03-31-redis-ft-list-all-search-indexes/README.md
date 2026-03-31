# How to Use FT._LIST in Redis to List All Search Indexes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Search

Description: Learn how to use FT._LIST in Redis to retrieve a list of all active RediSearch indexes for inventory, monitoring, and automation scripts.

---

`FT._LIST` returns the names of all RediSearch indexes that currently exist on the Redis server. It is a simple but useful inventory command for understanding what search indexes are deployed, verifying that indexes were created successfully, and driving automation scripts that need to iterate over all indexes.

## Note on the Name

The underscore prefix (`_LIST`) is intentional - in Redis, commands prefixed with `_` are internal or experimental. `FT._LIST` is stable and widely used, but the naming convention signals it was added informally. Some clients may require you to call it via `execute_command`.

## Syntax

```text
FT._LIST
```

No arguments. Returns an array of index name strings.

## Basic Example

```bash
# Create several indexes
FT.CREATE products ON HASH PREFIX 1 product: SCHEMA name TEXT price NUMERIC
FT.CREATE users ON HASH PREFIX 1 user: SCHEMA username TEXT email TEXT
FT.CREATE articles ON HASH PREFIX 1 article: SCHEMA title TEXT body TEXT

# List all indexes
FT._LIST
```

Response:

```text
1) "products"
2) "users"
3) "articles"
```

## Empty Result

If no indexes exist:

```bash
FT._LIST
```

Response:

```text
(empty array)
```

## Python Example: Listing All Indexes

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

indexes = r.execute_command("FT._LIST")
print(f"Found {len(indexes)} search index(es):")
for idx in indexes:
    print(f"  - {idx}")
```

## Automating Index Inspection

Combine `FT._LIST` with `FT.INFO` to generate a full report of all indexes:

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def index_report():
    indexes = r.execute_command("FT._LIST")
    print(f"Total indexes: {len(indexes)}")
    print("-" * 40)

    for idx in indexes:
        info = r.execute_command("FT.INFO", idx)
        info_dict = dict(zip(info[::2], info[1::2]))
        num_docs = info_dict.get("num_docs", "unknown")
        index_opts = info_dict.get("index_options", [])
        print(f"Index: {idx}")
        print(f"  Documents: {num_docs}")
        print(f"  Options: {index_opts}")
        print()

index_report()
```

## Deployment Verification Script

Use `FT._LIST` to verify all expected indexes exist after deployment:

```python
import redis
import sys

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

REQUIRED_INDEXES = ["products", "users", "articles", "orders"]

existing = set(r.execute_command("FT._LIST"))
missing = [idx for idx in REQUIRED_INDEXES if idx not in existing]

if missing:
    print(f"ERROR: Missing required indexes: {missing}")
    sys.exit(1)
else:
    print(f"All {len(REQUIRED_INDEXES)} required indexes are present")
```

## Cleanup Script: Drop All Indexes

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

# Drop all indexes (for test environment cleanup)
indexes = r.execute_command("FT._LIST")
for idx in indexes:
    r.execute_command("FT.DROPINDEX", idx)
    print(f"Dropped index: {idx}")
print(f"Removed {len(indexes)} index(es)")
```

## Aliases Are Not Listed

`FT._LIST` returns only actual index names, not aliases. Aliases created with `FT.ALIASADD` are resolved through their target index but do not appear as separate entries in `FT._LIST`.

## Summary

`FT._LIST` is the simplest way to enumerate all RediSearch indexes on a Redis server. It is particularly valuable in deployment scripts, monitoring tools, and administrative tasks where you need to dynamically discover or validate the set of active search indexes.
