# How Redis Dict (Hash Table) Implementation Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Hash Table, Dict, Internals, Data Structures, Rehashing

Description: Understand how Redis implements its internal hash table (dict), including incremental rehashing, collision resolution, and how it underlies key-value storage and Hash data types.

---

## What Is Redis Dict

The `dict` (dictionary) is the fundamental data structure in Redis. It is used for:
- The main keyspace (mapping key names to values)
- Hash data type (when using hashtable encoding)
- Set data type (when using hashtable encoding)
- Command table (command name to handler mapping)
- Pub/Sub subscription tracking

## Internal Structure

```c
// Simplified from redis/src/dict.h
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;  // linked list for collision chaining
} dictEntry;

typedef struct dictht {
    dictEntry **table;     // array of pointers to bucket chains
    unsigned long size;    // number of buckets (always power of 2)
    unsigned long sizemask; // size - 1 (for hash & sizemask = index)
    unsigned long used;    // number of stored key-value pairs
} dictht;

typedef struct dict {
    dictType *type;
    dictht ht[2];          // TWO hash tables for incremental rehashing
    long rehashidx;        // -1 if not rehashing, otherwise bucket index
    unsigned pauserehash;  // >0 if rehashing is paused
} dict;
```

## Collision Resolution: Chaining

Redis uses separate chaining with linked lists for hash collisions:

```text
Bucket 0: NULL
Bucket 1: [key1:val1] -> [key5:val5] -> NULL  (collision)
Bucket 2: [key2:val2] -> NULL
Bucket 3: [key3:val3] -> [key7:val7] -> NULL  (collision)
...
```

New entries are inserted at the head of the chain (O(1) insertion).

## Hash Function

Redis uses SipHash-1-2 by default (since Redis 4.0) for protection against hash flooding attacks:

```c
// Redis uses a random seed per-process to prevent hash collision attacks
uint64_t dictGenHashFunction(const void *key, size_t len) {
    return siphash(key, len, dict_hash_function_seed);
}

// For integer keys (faster)
uint64_t dictGenCaseHashFunction(const char *buf, size_t len);
```

## Load Factor and Rehashing Threshold

Redis triggers rehashing when:
- Normal operation: load factor >= 1 (more entries than buckets)
- With active child process: load factor >= 5 (to avoid copy-on-write overhead during BGSAVE)

```bash
# Monitor rehashing status
redis-cli INFO stats | grep rehash

# Check keyspace size and hash table info
redis-cli DEBUG RELOAD  # forces rehash
redis-cli INFO keyspace
```

## Incremental Rehashing (Progressive Rehashing)

Redis never rehashes all at once (which would block the server). Instead, it uses incremental rehashing:

```text
State: Not rehashing (rehashidx = -1)
  dict.ht[0] = active hash table
  dict.ht[1] = empty

Trigger: load factor >= 1
  dict.ht[1] = new hash table (2x size of ht[0])
  rehashidx = 0

During rehashing:
  On every dict operation (GET/SET/DEL):
    1. Move up to N entries from ht[0][rehashidx] to ht[1]
    2. rehashidx++
  Also: background timer moves 100 entries per 1ms

Completion:
  When ht[0] is empty:
    ht[0] = ht[1]
    ht[1] = NULL
    rehashidx = -1
```

## How Rehashing Affects Operations

During rehashing, both hash tables are active:

```text
GET key:
  1. Look in ht[0] at key's bucket
  2. If not found, look in ht[1]

SET key:
  1. Always insert into ht[1]

DEL key:
  1. Search both ht[0] and ht[1]

Iteration (SCAN):
  1. Must iterate both tables
  2. Handles duplicates that may appear during rehash
```

## Observing Dict Behavior

```bash
# Force a rehash by adding many keys
for i in $(seq 1 10000); do
  redis-cli SET "key:$i" "value:$i" > /dev/null
done

# Check memory and stats
redis-cli INFO memory | grep -E "used_memory|mem_fragmentation"

# DEBUG RELOAD forces immediate rehash (use with caution in production)
redis-cli DEBUG RELOAD

# Check dict stats via INFO
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
```

## Dict Shrinking

Redis also shrinks hash tables when the load factor drops below 0.1:

```text
Shrink condition: used < size * 0.1 (10% full)
Triggered by: HDEL, DEL commands reducing element count
Process: Same incremental rehash with a smaller ht[1]
```

## The Keyspace as a Dict

The global keyspace is itself a `dict`:

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: process.env.REDIS_HOST || 'localhost' });

// DBSIZE queries the dict.used field (O(1))
const keyCount = await redis.dbsize();
console.log(`Keys in database: ${keyCount}`);

// EXISTS checks the dict (O(1))
const exists = await redis.exists('mykey');

// INFO keyspace shows per-database dict stats
const info = await redis.info('keyspace');
console.log(info);
// db0:keys=1000,expires=50,avg_ttl=60000
```

## Impact on SCAN Cursor Design

The SCAN cursor is actually a reverse binary cursor that accounts for rehashing:

```text
Normal cursor: 0, 1, 2, 3, 4... (sequential)
Redis cursor:  0, 8, 4, 12, 2, 10... (reverse bit iteration)

Reverse iteration ensures:
- All existing keys are returned even if rehashing changes table size
- No key is missed when the table grows from 8 to 16 buckets mid-scan
- Some keys may be returned twice (duplicates expected)
```

## Summary

Redis dict is a chained hash table with incremental rehashing that ensures no single operation blocks the server during table resizing. The dual-table design (ht[0] and ht[1]) allows rehashing to proceed gradually across many operations, with writes always going to the new table and reads checking both. Understanding incremental rehashing explains why SCAN may return duplicates and why memory usage temporarily doubles during large hash table growth.
