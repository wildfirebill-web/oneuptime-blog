# Why You Should Not Use DEL for Large Keys in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Memory, Anti-Pattern

Description: Learn why DEL blocks Redis while freeing large keys and how to use UNLINK for async non-blocking deletion of large lists, sets, and hashes.

---

DEL removes a key synchronously. For small keys, this is instant. For large keys - a list with a million elements, a hash with hundreds of thousands of fields, a set with massive cardinality - DEL can block Redis for milliseconds or even seconds while it frees memory. In production, this causes latency spikes across your entire application.

## Why DEL Blocks on Large Keys

Redis is single-threaded. When you DEL a key, Redis must free all memory associated with that key before processing the next command. A list with 1 million entries means 1 million memory deallocations:

```bash
# Creating a large key
RPUSH big:list $(python3 -c "print(' '.join(['item'] * 1000000))")

# This blocks Redis while freeing 1M entries:
DEL big:list
# Execution time: 50-200ms depending on server
```

During those milliseconds, every other command waits.

## The Fix: UNLINK

UNLINK was introduced in Redis 4.0. It unlinks the key from the keyspace immediately (making it invisible) but frees the memory asynchronously in a background thread:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Wrong: blocks for large keys
# r.delete("big:list")

# Right: async deletion, non-blocking
def safe_delete(key: str):
    r.unlink(key)

# Bulk safe delete
def safe_delete_many(keys: list[str]):
    if keys:
        r.unlink(*keys)
```

## Identifying Large Keys Before Deletion

Before deleting any key, check its size:

```python
def get_key_size(key: str) -> dict:
    key_type = r.type(key)
    size = 0

    if key_type == "list":
        size = r.llen(key)
    elif key_type == "set":
        size = r.scard(key)
    elif key_type == "zset":
        size = r.zcard(key)
    elif key_type == "hash":
        size = r.hlen(key)
    elif key_type == "string":
        size = r.strlen(key)

    return {"type": key_type, "size": size}

LARGE_KEY_THRESHOLD = 10000

def delete_key_safely(key: str):
    info = get_key_size(key)
    if info["size"] > LARGE_KEY_THRESHOLD:
        print(f"Large key detected ({info['size']} elements) - using UNLINK")
        r.unlink(key)
    else:
        r.delete(key)
```

## Incrementally Deleting Very Large Keys

For keys with millions of elements and limited memory headroom, delete in batches before UNLINKing:

```python
def delete_large_list(key: str, batch_size: int = 10000):
    """Delete a large list incrementally to avoid memory spikes."""
    while r.llen(key) > 0:
        # Remove batch_size elements from the tail
        r.ltrim(key, 0, -batch_size - 1)

    r.unlink(key)

def delete_large_hash(key: str, batch_size: int = 1000):
    """Delete a large hash incrementally using HSCAN."""
    cursor = 0
    while True:
        cursor, fields = r.hscan(key, cursor, count=batch_size)
        if fields:
            r.hdel(key, *fields.keys())
        if cursor == 0:
            break
    r.unlink(key)

def delete_large_set(key: str, batch_size: int = 1000):
    """Delete a large set incrementally using SSCAN."""
    cursor = 0
    while True:
        cursor, members = r.sscan(key, cursor, count=batch_size)
        if members:
            r.srem(key, *members)
        if cursor == 0:
            break
    r.unlink(key)
```

## Enabling Lazy Freeing Globally

Configure Redis to use async deletion for all internal deletions too:

```bash
# redis.conf
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes
lazyfree-lazy-user-flush yes
```

## Summary

DEL is a blocking operation that freezes Redis while freeing all memory for a key. For any key with thousands or more elements, use UNLINK instead to free memory asynchronously in a background thread. Enable global lazy freeing in redis.conf so that expiration and eviction also avoid blocking the main thread.
