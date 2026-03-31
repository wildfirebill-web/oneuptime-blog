# How to Use MEMORY USAGE in Redis to Check Key Memory Size

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Memory, Performance, Optimization, Keys

Description: Learn how to use MEMORY USAGE in Redis to measure the exact memory footprint of individual keys, helping you identify memory-heavy data structures.

---

## What Is MEMORY USAGE?

`MEMORY USAGE` returns the number of bytes that a Redis key and its value consume in memory. It includes not just the raw value but also the overhead from the data structure, key name storage, and any associated bookkeeping. This makes it more accurate than estimating based on value length alone.

## Basic Syntax

```text
MEMORY USAGE key [SAMPLES count]
```

Parameters:
- `key` - the key to measure
- `SAMPLES count` - for aggregate types (lists, sets, hashes, sorted sets), the number of elements to sample for estimation (default is 5, use 0 to sample all)

Returns the number of bytes used, or a nil reply if the key does not exist.

## Measuring Basic Types

```bash
# String
SET greeting "Hello, World!"
MEMORY USAGE greeting
# Returns: 56 (bytes, including key overhead)

# Integer stored as string
SET counter 1000000
MEMORY USAGE counter
# Returns: 52

# Large string
SET bigval "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
MEMORY USAGE bigval
# Returns: 104
```

## Measuring Aggregate Types

```bash
# Hash
HSET user:1 name "Alice" age "30" email "alice@example.com"
MEMORY USAGE user:1
# Returns: 120

# List
RPUSH events "login" "logout" "purchase" "view"
MEMORY USAGE events
# Returns: 128

# Set
SADD tags "redis" "database" "cache" "nosql"
MEMORY USAGE tags
# Returns: 168

# Sorted Set
ZADD leaderboard 100 "Alice" 95 "Bob" 88 "Charlie"
MEMORY USAGE leaderboard
# Returns: 176
```

## Using SAMPLES for Large Collections

For large collections, `MEMORY USAGE` samples a subset of elements to estimate the total. Increase the sample count for more accuracy at the cost of performance:

```bash
# Large hash with 1000 fields
# Default sampling (5 elements)
MEMORY USAGE big:hash
# Returns: ~45000 (estimate)

# Sample all elements for exact measurement
MEMORY USAGE big:hash SAMPLES 0
# Returns: 45230 (exact)

# Balance accuracy and speed with more samples
MEMORY USAGE big:hash SAMPLES 100
```

## Comparing Encoding Impact

Redis uses different internal encodings for the same data type. Memory usage varies by encoding:

```bash
# Small hash (uses ziplist/listpack - compact)
HSET small:hash f1 v1 f2 v2
MEMORY USAGE small:hash
# Returns: 72 (compact encoding)

# Large hash (converted to hashtable)
# After adding >128 fields or fields with values >64 bytes
MEMORY USAGE large:hash
# Returns: 1280 (hashtable encoding - more overhead)
```

## Practical: Finding Memory-Heavy Keys

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Scan all keys and find the largest ones
top_keys = []

cursor = 0
while True:
    cursor, keys = r.scan(cursor, count=100)
    for key in keys:
        usage = r.memory_usage(key)
        if usage:
            top_keys.append((key, usage))
    if cursor == 0:
        break

# Sort by memory usage descending
top_keys.sort(key=lambda x: x[1], reverse=True)

print("Top 10 memory-hungry keys:")
for key, size in top_keys[:10]:
    print(f"  {key}: {size:,} bytes ({size / 1024:.1f} KB)")
```

## Comparing Data Structure Options

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

user_data = {'name': 'Alice', 'age': 30, 'email': 'alice@example.com', 'city': 'NYC'}

# Option 1: Store as JSON string
r.set('user:json:1', json.dumps(user_data))
json_usage = r.memory_usage('user:json:1')

# Option 2: Store as Hash
r.hset('user:hash:1', mapping=user_data)
hash_usage = r.memory_usage('user:hash:1')

print(f"JSON string: {json_usage} bytes")
print(f"Hash: {hash_usage} bytes")
print(f"Difference: {abs(json_usage - hash_usage)} bytes")
```

## Summary

`MEMORY USAGE` is an essential Redis debugging tool that reveals the true memory footprint of individual keys, including encoding overhead and bookkeeping costs. Use it to identify unexpectedly large keys, compare encoding strategies, and optimize memory-heavy data structures. Combine it with `SCAN` for a full keyspace audit, and use `SAMPLES 0` when you need exact measurements for large aggregate types.
