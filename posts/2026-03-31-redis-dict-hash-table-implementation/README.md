# How Redis Dict (Hash Table) Implementation Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Internals, Hash Table, Dictionary, Data Structures

Description: Explore how Redis implements its internal dictionary using a double-hash-table structure with incremental rehashing to avoid blocking the event loop.

---

## What Is the Redis Dict

The `dict` (dictionary) is the central data structure in Redis. It is used for:

- The global keyspace (all keys in a database)
- Hash data type values (when using hashtable encoding)
- The command table (mapping command names to handlers)
- Various internal registries

The dict is a chained hash table implemented in `dict.c` in the Redis source.

## Dict Structure Overview

Each dict contains two hash tables (`ht[0]` and `ht[1]`):

```text
dict
+---------------------+
| ht[0]               |  <- active hash table
|   size: 8           |
|   used: 5           |
|   table: [ptr, ...]  |
+---------------------+
| ht[1]               |  <- used only during rehashing
|   size: 0           |
|   used: 0           |
|   table: NULL        |
+---------------------+
| rehashidx: -1        |  <- -1 means not currently rehashing
+---------------------+
```

Each slot in the table array is a pointer to a linked list of `dictEntry` nodes:

```text
table[3] -> [key:"user:1", val:..., next:->] -> [key:"user:9", val:..., next:NULL]
```

## Hash Function

Redis uses SipHash-1-2 as its hash function (since Redis 4.0). Prior versions used MurmurHash2. SipHash provides protection against hash collision DoS attacks.

```text
slot = siphash(key) & (size - 1)
```

The table size is always a power of 2, so the modulo operation reduces to a bitwise AND.

## Load Factor and Rehashing Threshold

The load factor is the ratio of stored entries to table slots:

```text
load_factor = ht[0].used / ht[0].size
```

Rehashing is triggered when:

- `load_factor >= 1` and no background save is running
- `load_factor >= 5` regardless of background save (forced rehash)

The new table size is the smallest power of 2 greater than `ht[0].used * 2`.

## Incremental Rehashing

A naive rehash would copy all entries at once, blocking the event loop for an unacceptable amount of time on large tables. Redis uses **incremental rehashing** instead.

When rehash is triggered:
1. `ht[1]` is allocated with the new size
2. `rehashidx` is set to 0 (the current bucket being migrated)
3. On each dict operation (GET, SET, DELETE), Redis migrates a small number of buckets from `ht[0]` to `ht[1]`
4. When all buckets are migrated, `ht[1]` becomes `ht[0]` and `ht[1]` is reset

```text
Step 1: rehashidx = 0
  Migrate bucket 0 from ht[0] to ht[1]

Step 2: rehashidx = 1
  Migrate bucket 1 from ht[0] to ht[1]

...

Step N: All buckets migrated
  ht[0] = ht[1]
  ht[1] = empty
  rehashidx = -1
```

During rehashing, lookups check both `ht[0]` and `ht[1]`. New keys are always inserted into `ht[1]`.

## Observing Rehashing in Action

You can observe rehash activity indirectly via `INFO stats`:

```bash
redis-cli INFO stats | grep rehash
```

```text
active_defrag_running:0
lazyfree_pending_objects:0
```

For a more direct view, use `DEBUG DICT-RESIZE-RATIO`:

```bash
redis-cli DEBUG SLEEP 0
redis-cli DBSIZE
```

You can also check how many keys are in each hash table via the `DEBUG OBJECT` command on a key:

```bash
redis-cli DEBUG OBJECT mykey
```

## Active Rehash Steps

Beyond per-operation rehashing, Redis calls `dictRehashMilliseconds()` in the server cron to perform up to 1ms of rehashing per 100ms interval. This accelerates rehashing when the database is lightly loaded.

```text
Every 100ms server cron:
  - Spend up to 1ms migrating buckets from ht[0] to ht[1]
  - Each step migrates 100 non-empty buckets
```

## Collision Handling

Redis uses separate chaining (linked lists) to handle hash collisions. Each bucket slot holds a pointer to a linked list of `dictEntry` nodes. In the worst case all keys hash to the same slot, reducing lookup to O(N). In practice SipHash produces well-distributed hashes.

```text
Bucket 5: [key="a", next] -> [key="b", next] -> [key="c", next=NULL]
```

## Memory Layout of dictEntry

Each entry stores:

```text
dictEntry {
  void *key;
  union {
    void *val;
    uint64_t u64;
    int64_t  s64;
    double   d;
  } v;
  struct dictEntry *next;
  void *metadata[];
}
```

Integer values are stored inline in the union (no separate heap allocation) when they fit in 64 bits.

## Shrinking the Hash Table

Redis also shrinks the table when the load factor drops below 0.1 (the dictionary becomes sparse). This frees unused memory:

```text
Shrink threshold: ht[0].used / ht[0].size < 0.1
New size: smallest power of 2 > ht[0].used
```

## Practical Implications

**1. Avoid mass deletion without compaction:**
Deleting thousands of keys leaves empty buckets. Redis will eventually shrink, but you can force it:

```bash
redis-cli DEBUG RELOAD
```

**2. Do not run KEYS during heavy rehash:**
During rehash, `KEYS` and similar full-scan commands must check both `ht[0]` and `ht[1]`, doubling the work.

**3. Monitor memory during rapid key growth:**
When a large number of keys are added quickly, the new `ht[1]` allocation doubles memory usage temporarily. Plan memory headroom accordingly.

## Summary

Redis implements its dict as a double hash table with incremental rehashing. When the load factor threshold is exceeded, Redis gradually migrates entries from the active table to a new, larger table across multiple operations, avoiding long blocking pauses. The SipHash function provides collision resistance, and separate chaining handles collisions efficiently. Understanding this mechanism helps you size your Redis instance correctly and avoid patterns that cause excessive rehashing overhead.
