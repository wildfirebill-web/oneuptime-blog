# How to Use Redis JSON (RedisJSON) in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, RedisJSON, JSON, redis-py

Description: Store, query, and update JSON documents in Redis using the RedisJSON module with redis-py's JSON client, including JSONPath queries and partial updates.

---

RedisJSON adds native JSON support to Redis, allowing you to store documents and query or update them using JSONPath expressions without fetching the entire document. This guide covers the Python API available through `redis-py`.

## Prerequisites

Install Redis Stack (which includes RedisJSON) or enable the module on your server:

```bash
docker run -d -p 6379:6379 redis/redis-stack-server:latest
```

Install the Python client:

```bash
pip install redis
```

## Storing a JSON Document

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

user = {
    "id": 1001,
    "name": "Alice",
    "email": "alice@example.com",
    "address": {
        "city": "San Francisco",
        "country": "US",
    },
    "tags": ["python", "redis"],
    "score": 9.5,
}

r.json().set("user:1001", "$", user)
print("Stored user document")
```

## Retrieving Documents

```python
# Get the entire document
doc = r.json().get("user:1001")
print(doc)  # Returns the Python dict

# Get a nested field using JSONPath
city = r.json().get("user:1001", "$.address.city")
print(city)  # ['San Francisco']

# Get multiple paths at once
result = r.json().get("user:1001", "$.name", "$.email")
print(result)
```

## Partial Updates

```python
# Update a single field
r.json().set("user:1001", "$.score", 9.8)

# Update a nested field
r.json().set("user:1001", "$.address.city", "New York")

# Append to an array
r.json().arrappend("user:1001", "$.tags", "kubernetes")

# Increment a numeric field
r.json().numincrby("user:1001", "$.score", 0.1)
print(r.json().get("user:1001", "$.score"))  # [9.9]
```

## Bulk Operations

```python
# Set multiple documents
docs = {
    "user:1002": {"name": "Bob", "score": 7.0, "tags": ["go"]},
    "user:1003": {"name": "Charlie", "score": 8.5, "tags": ["rust"]},
}
for key, doc in docs.items():
    r.json().set(key, "$", doc)

# Get the same field from multiple keys
names = r.json().mget(["user:1001", "user:1002", "user:1003"], "$.name")
print(names)  # [['Alice'], ['Bob'], ['Charlie']]
```

## Getting Document Structure

```python
# Get the type of a path
print(r.json().type("user:1001", "$.tags"))    # ['array']
print(r.json().type("user:1001", "$.score"))   # ['number']

# Get array length
print(r.json().arrlen("user:1001", "$.tags"))  # [3]

# Get object keys at a path
print(r.json().objkeys("user:1001", "$.address"))  # [['city', 'country']]
```

## Deleting Fields

```python
# Delete a specific field
r.json().delete("user:1001", "$.address.country")

# Delete the entire document
r.json().delete("user:1001")
```

## Combining with RediSearch

With Redis Stack, you can index JSON fields and run full-text or numeric queries:

```bash
redis-cli FT.CREATE users ON JSON PREFIX 1 user: SCHEMA $.name AS name TEXT $.score AS score NUMERIC
```

Then query from Python using the `redis.commands.search` API.

## Summary

RedisJSON lets you store entire JSON documents in Redis and update individual fields using JSONPath without round-tripping the full document. The `redis-py` JSON client exposes commands like `json().set()`, `json().get()`, `json().arrappend()`, and `json().numincrby()` for efficient document manipulation directly from Python.
