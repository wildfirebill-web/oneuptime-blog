# How to Use Listpack Encoding in Redis 7.0+

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Listpack, Memory Optimization, Data Structures, Redis 7

Description: Learn how Redis 7.0 replaced ziplist with listpack for compact storage, how encoding thresholds work, and how to tune them for memory efficiency.

---

## What Is Listpack

Listpack is a compact, sequential memory layout for small collections. Redis 7.0 replaced the older ziplist encoding with listpack across all data types: hashes, sets, sorted sets, and lists.

Listpack stores entries end-to-end in a flat memory buffer. Each entry contains its own length, so the structure can be traversed both forward and backward without a separate index.

```text
Listpack memory layout:
+--------+--------+--------+--------+--------+
| HEADER | ENTRY1 | ENTRY2 | ENTRY3 | END    |
+--------+--------+--------+--------+--------+
  6 bytes  var     var     var      1 byte
```

The key advantage over ziplist is that listpack entries do not store a "previous entry length", eliminating cascade update issues when entries change size.

## Which Data Types Use Listpack

In Redis 7.0+, listpack is used as the compact encoding for:

| Data Type   | Compact Encoding | Full Encoding  |
|-------------|------------------|----------------|
| Hash        | listpack         | hashtable      |
| Set         | listpack         | hashtable      |
| Sorted Set  | listpack         | skiplist       |
| List        | listpack         | quicklist      |

## Checking the Current Encoding

Use `OBJECT ENCODING` to see whether a key is using listpack:

```bash
redis-cli HSET user:1 name "Alice" age "30"
redis-cli OBJECT ENCODING user:1
```

```text
"listpack"
```

Once the collection grows beyond configured thresholds, it promotes to the full encoding:

```bash
# Add many fields to exceed threshold
for i in $(seq 1 200); do
  redis-cli HSET user:1 "field$i" "value$i"
done

redis-cli OBJECT ENCODING user:1
```

```text
"hashtable"
```

## Listpack Thresholds by Data Type

### Hash

```bash
redis-cli CONFIG GET hash-max-listpack-entries
redis-cli CONFIG GET hash-max-listpack-value
```

```text
hash-max-listpack-entries: 128
hash-max-listpack-value:   64
```

- `hash-max-listpack-entries`: max number of fields before switching to hashtable
- `hash-max-listpack-value`: max byte size of any field or value before switching

### Sorted Set

```bash
redis-cli CONFIG GET zset-max-listpack-entries
redis-cli CONFIG GET zset-max-listpack-value
```

```text
zset-max-listpack-entries: 128
zset-max-listpack-value:   64
```

### Set (integer or small string sets)

```bash
redis-cli CONFIG GET set-max-listpack-entries
redis-cli CONFIG GET set-max-listpack-value
```

```text
set-max-listpack-entries: 128
set-max-listpack-value:   64
```

### List

Lists use listpack for very small lists within quicklist nodes:

```bash
redis-cli CONFIG GET list-max-listpack-size
```

```text
list-max-listpack-size: -2
```

Negative values refer to size limits: `-2` means each quicklist node is at most 8KB.

## Tuning Thresholds for Memory Savings

If your hashes consistently have fewer than 256 fields and values under 128 bytes, you can increase the thresholds to keep more hashes in listpack:

```bash
redis-cli CONFIG SET hash-max-listpack-entries 256
redis-cli CONFIG SET hash-max-listpack-value 128
```

To make this permanent, add to `redis.conf`:

```text
hash-max-listpack-entries 256
hash-max-listpack-value 128
zset-max-listpack-entries 256
zset-max-listpack-value 128
set-max-listpack-entries 256
set-max-listpack-value 128
```

## Memory Savings Example

Consider 1 million hashes each with 10 fields:

- **hashtable encoding**: ~180 bytes per hash = ~172 MB
- **listpack encoding**: ~90 bytes per hash = ~86 MB

Listpack roughly halves the memory footprint for small hashes.

Check actual per-key memory:

```bash
redis-cli MEMORY USAGE user:1
```

```text
(integer) 96
```

## Converting Existing Keys to Listpack

If you changed thresholds after keys were already promoted to full encodings, existing keys do not automatically downgrade. You need to re-insert them.

A migration script in Python:

```python
import redis

r = redis.Redis(host='localhost', port=6379)
cursor = 0

while True:
    cursor, keys = r.scan(cursor=cursor, match='user:*', count=200)
    for key in keys:
        key_str = key.decode()
        if r.object_encoding(key_str) == b'hashtable':
            data = r.hgetall(key_str)
            ttl = r.ttl(key_str)
            pipe = r.pipeline()
            pipe.delete(key_str)
            pipe.hset(key_str, mapping={k.decode(): v.decode() for k, v in data.items()})
            if ttl > 0:
                pipe.expire(key_str, ttl)
            pipe.execute()
    if cursor == 0:
        break
```

## Listpack vs Ziplist: Key Differences

| Feature               | Ziplist         | Listpack        |
|-----------------------|-----------------|-----------------|
| Redis version         | < 7.0           | 7.0+            |
| Cascade updates       | Yes (slow)      | No              |
| Memory layout         | Sequential      | Sequential      |
| Entry size field      | prevlen + len   | len only        |
| Performance under mod | O(N) worst case | O(1) per entry  |

## Summary

Redis 7.0 replaced ziplist with listpack as the compact encoding for hashes, sets, sorted sets, and small lists. Listpack eliminates the cascade update problem of ziplist while maintaining similar memory efficiency. By tuning the `max-listpack-entries` and `max-listpack-value` thresholds for each data type, you can keep more keys in the memory-efficient listpack encoding and significantly reduce your Redis memory footprint.
