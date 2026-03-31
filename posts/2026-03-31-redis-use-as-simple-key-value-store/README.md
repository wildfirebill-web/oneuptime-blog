# How to Use Redis as a Simple Key-Value Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Key-Value, Beginner, String, Cache

Description: Learn how to use Redis as a simple key-value store - storing, retrieving, updating, and deleting values with practical examples.

---

The most fundamental use of Redis is as a key-value store. Every key is a string, and the value can be a string, number, JSON blob, or any other data. This guide covers the basics.

## Setting and Getting Values

```bash
# Set a key
SET greeting "Hello, World!"

# Get a key
GET greeting
# "Hello, World!"

# Key doesn't exist returns nil
GET nonexistent
# (nil)
```

## Storing Different Value Types

Redis strings can hold any text or binary data:

```bash
# Plain text
SET username "alice"

# Number (stored as string, but supports numeric operations)
SET visit_count 0

# JSON object
SET user:42 '{"id":42,"name":"Alice","email":"alice@example.com"}'

# Binary data (images, serialized objects, etc.)
# Up to 512 MB per value
```

## Checking and Deleting Keys

```bash
# Check if key exists (returns 1 = yes, 0 = no)
EXISTS username
EXISTS nonexistent

# Delete a key
DEL username
# (integer) 1  -> number of keys deleted

# Delete multiple keys at once
DEL key1 key2 key3
```

## Updating Values

```bash
# Overwrite an existing value
SET username "bob"    # replaces "alice"

# Only set if key does NOT exist
SET username "carol" NX
# (nil) -> no-op because username already exists

# Only set if key DOES exist
SET username "dave" XX
# OK -> succeeds because username exists
```

## Working with Numbers

```bash
SET counter 0

# Increment by 1
INCR counter
# (integer) 1

# Decrement by 1
DECR counter
# (integer) 0

# Add a specific amount
INCRBY counter 10
# (integer) 10

# Add a float
INCRBYFLOAT price 4.99
```

## Setting Multiple Keys at Once

```bash
# Set multiple keys in one command (atomic)
MSET first_name "Alice" last_name "Smith" city "London"

# Get multiple keys in one command
MGET first_name last_name city
# 1) "Alice"
# 2) "Smith"
# 3) "London"
```

## Key Naming Conventions

Redis keys are arbitrary strings, but a common convention is to use colons as namespace separators:

```bash
SET user:42:name "Alice"
SET user:42:email "alice@example.com"
SET session:abc123 "user_id=42"
SET product:99:price "29.99"
SET stats:2026-03-31:page_views 150
```

## Practical Example in Python

```python
import redis
import json

r = redis.Redis(host='127.0.0.1', port=6379, decode_responses=True)

# Store a user profile as JSON
user = {"id": 42, "name": "Alice", "email": "alice@example.com"}
r.set("user:42", json.dumps(user))

# Retrieve and deserialize
raw = r.get("user:42")
user_data = json.loads(raw)
print(user_data["name"])  # Alice

# Counter example
r.set("visits", 0)
r.incr("visits")
r.incr("visits")
print(r.get("visits"))  # 2
```

## Summary

Redis as a key-value store is as simple as SET and GET. Use colon-separated naming for organization, MSET/MGET for batch operations, INCR/DECR for counters, and NX/XX options for conditional writes. For structured data, serialize to JSON before storing in a Redis string.
