# How to Use Listpack Encoding in Redis 7.0+

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Listpack, Memory Optimization, Encoding, Redis 7.0

Description: Understand Redis 7.0 listpack encoding, how it replaces ziplist for compact memory storage, and how to tune thresholds for optimal memory efficiency.

---

## What Is Listpack

Listpack is a compact sequential data structure introduced in Redis 5.0 as a successor to the ziplist encoding. Starting in Redis 7.0, listpack fully replaces ziplist as the compact encoding for small collections.

Listpack uses less memory than ziplist due to a simpler internal structure:
- No cascading update problem that affected ziplist
- Simpler encoding for integers and short strings
- Used by Hashes, Lists, Sets, and Sorted Sets for small collections

## When Redis Uses Listpack

Redis automatically uses listpack encoding when a collection is below the configured thresholds:

```bash
# Hashes: listpack used when entries <= 128 AND values <= 64 bytes
redis-cli CONFIG GET hash-max-listpack-entries
redis-cli CONFIG GET hash-max-listpack-value

# Lists: listpack used for small lists (per node)
redis-cli CONFIG GET list-max-listpack-size

# Sets: listpack used when entries <= 128 AND values <= 64 bytes
redis-cli CONFIG GET set-max-listpack-entries
redis-cli CONFIG GET set-max-listpack-value

# Sorted Sets: listpack used when entries <= 128 AND values <= 64 bytes
redis-cli CONFIG GET zset-max-listpack-entries
redis-cli CONFIG GET zset-max-listpack-value
```

## Checking Current Encoding

```bash
# Check encoding of a specific key
redis-cli OBJECT ENCODING mykey

# Listpack encoding shown as "listpack"
# When threshold exceeded, switches to:
# - "hashtable" for Hashes and Sets
# - "skiplist" for Sorted Sets
# - "quicklist" for Lists
```

## Hash Listpack Configuration

```bash
# Default thresholds (Redis 7.0+)
CONFIG SET hash-max-listpack-entries 128
CONFIG SET hash-max-listpack-value 64

# Verify encoding
127.0.0.1:6379> HSET smallhash field1 value1 field2 value2
127.0.0.1:6379> OBJECT ENCODING smallhash
"listpack"

# Add many fields to trigger conversion
127.0.0.1:6379> HSET bighash $(seq 1 150 | awk '{printf "field%d val%d ", $1, $1}')
127.0.0.1:6379> OBJECT ENCODING bighash
"hashtable"
```

## Sorted Set Listpack Configuration

```bash
CONFIG SET zset-max-listpack-entries 128
CONFIG SET zset-max-listpack-value 64

# Small sorted set uses listpack
127.0.0.1:6379> ZADD scores 100 "alice" 200 "bob" 150 "charlie"
127.0.0.1:6379> OBJECT ENCODING scores
"listpack"
```

## Set Listpack Configuration

```bash
# Redis 7.2+ allows sets to use listpack
CONFIG SET set-max-listpack-entries 128
CONFIG SET set-max-listpack-value 64

# Small set with non-integer members uses listpack
127.0.0.1:6379> SADD tags "redis" "caching" "database"
127.0.0.1:6379> OBJECT ENCODING tags
"listpack"

# Integer sets use intset encoding (even more compact)
127.0.0.1:6379> SADD numbers 1 2 3 4 5
127.0.0.1:6379> OBJECT ENCODING numbers
"intset"
```

## Tuning Thresholds for Memory Optimization

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: process.env.REDIS_HOST || 'localhost' });

async function analyzeEncodings(pattern = '*') {
  let cursor = '0';
  const encodingCounts = {};

  do {
    const [newCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = newCursor;

    for (const key of keys) {
      const encoding = await redis.object('ENCODING', key);
      encodingCounts[encoding] = (encodingCounts[encoding] ?? 0) + 1;
    }
  } while (cursor !== '0');

  console.table(encodingCounts);
}

await analyzeEncodings('user:*');
```

## Memory Savings from Listpack

Compare memory usage between listpack and hashtable:

```bash
# Create a small hash (listpack)
redis-cli HSET small-hash name "Alice" age "30" city "NYC"
redis-cli MEMORY USAGE small-hash
# Output: ~120 bytes

# Create with large value forcing hashtable
redis-cli HSET big-hash name "Alice" description "$(python3 -c "print('x' * 65)")"
redis-cli OBJECT ENCODING big-hash
# "hashtable"
redis-cli MEMORY USAGE big-hash
# Output: ~400 bytes (3x more)
```

## Optimal Configuration for Memory Efficiency

```bash
# Maximize listpack usage for small objects
# (adjust based on your actual data sizes)
CONFIG SET hash-max-listpack-entries 256
CONFIG SET hash-max-listpack-value 128
CONFIG SET zset-max-listpack-entries 256
CONFIG SET zset-max-listpack-value 128
CONFIG SET set-max-listpack-entries 256
CONFIG SET set-max-listpack-value 128

# Save to redis.conf
CONFIG REWRITE
```

## Listpack vs Ziplist Migration

In Redis 7.0+, ziplist has been fully replaced by listpack. When upgrading:

```bash
# Check Redis version
redis-cli INFO server | grep redis_version

# After upgrade to 7.0+, existing ziplist-encoded keys
# will continue to work but new small collections use listpack

# Force re-encoding by reading and rewriting the data
redis-cli DUMP mykey | redis-cli RESTORE mykey 0 -
```

## Summary

Listpack encoding in Redis 7.0+ is the compact storage format for small collections, replacing ziplist with a more efficient and simpler structure. Redis automatically uses listpack when collections are below the configured entry count and value size thresholds. Tune `hash-max-listpack-entries`, `zset-max-listpack-entries`, and `set-max-listpack-entries` to maximize the number of keys stored in the memory-efficient listpack format.
