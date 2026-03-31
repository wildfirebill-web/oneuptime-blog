# How to Use Redis CLI --scan for Safe Key Enumeration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis Cli, SCAN, Key Enumeration, Production

Description: Use redis-cli --scan to safely enumerate Redis keys in production without blocking the server, unlike the dangerous KEYS command which blocks all operations.

---

## Why KEYS Is Dangerous in Production

The `KEYS pattern` command scans all keys in O(N) time and blocks the Redis server for the entire duration. On a database with millions of keys, this can cause:

- Server-wide latency spikes of seconds or more
- Timeout errors in all connected clients
- Cascading failures in dependent services

**Never use KEYS in production.** Use SCAN instead.

## The --scan Flag in redis-cli

The `--scan` flag makes redis-cli use the `SCAN` command iteratively under the hood, fetching keys in small batches without blocking the server:

```bash
# List all keys (equivalent to KEYS * but safe)
redis-cli --scan

# Match a specific pattern
redis-cli --scan --pattern "user:*"

# Match with a specific database
redis-cli -n 2 --scan --pattern "session:*"

# Count matching keys
redis-cli --scan --pattern "cache:*" | wc -l
```

## Controlling Batch Size with COUNT

The `COUNT` option hints to Redis how many keys to return per SCAN iteration. Larger values reduce round trips but use slightly more server CPU per iteration:

```bash
# Default count (10)
redis-cli --scan --pattern "user:*"

# Larger batch size for faster enumeration on large datasets
redis-cli --scan --pattern "product:*" --count 1000
```

## Practical Use Cases

### Delete keys matching a pattern

```bash
# Safe pattern-based deletion using xargs
redis-cli --scan --pattern "temp:*" | xargs -L 100 redis-cli DEL

# Slightly faster version using xargs with multiple args per command
redis-cli --scan --pattern "session:expired:*" | xargs redis-cli DEL
```

### Find keys without TTL

```bash
# Get all keys and check their TTL
redis-cli --scan --pattern "cache:*" | while read key; do
  ttl=$(redis-cli TTL "$key")
  if [ "$ttl" = "-1" ]; then
    echo "No TTL: $key"
  fi
done
```

### Export key names to a file

```bash
redis-cli --scan --pattern "product:*" > product_keys.txt
echo "Found $(wc -l < product_keys.txt) product keys"
```

## Using SCAN Programmatically

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: process.env.REDIS_HOST || 'localhost' });

async function* scanKeys(pattern = '*', count = 100) {
  let cursor = '0';
  do {
    const [newCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', count);
    cursor = newCursor;
    for (const key of keys) {
      yield key;
    }
  } while (cursor !== '0');
}

// Usage
let total = 0;
for await (const key of scanKeys('user:*', 200)) {
  total++;
  // Process key
}
console.log(`Found ${total} keys`);
```

## Type-Specific SCAN Commands

Each Redis data type has its own SCAN variant for iterating over its members:

```bash
# HSCAN - iterate hash fields
redis-cli HSCAN myhash 0 MATCH "field:*" COUNT 50

# SSCAN - iterate set members
redis-cli SSCAN myset 0 COUNT 100

# ZSCAN - iterate sorted set members with scores
redis-cli ZSCAN myzset 0 MATCH "user:*" COUNT 100

# LRANGE is used for lists (no SCAN equivalent)
redis-cli LRANGE mylist 0 99  # First 100 elements
```

## Counting Keys Without Blocking

```bash
# DBSIZE returns total key count instantly (O(1))
redis-cli DBSIZE

# INFO keyspace shows per-database key counts
redis-cli INFO keyspace
# Output: db0:keys=1000000,expires=50000,avg_ttl=300000
```

## SCAN Guarantees and Limitations

SCAN provides these guarantees:
- A key that was present for the full iteration will be returned
- A key added or deleted during iteration may or may not be returned
- Duplicate keys may be returned (always deduplicate results)

```javascript
// Deduplicate scan results
async function getAllKeys(pattern) {
  const seen = new Set();

  for await (const key of scanKeys(pattern, 200)) {
    seen.add(key);
  }

  return [...seen];
}
```

## Performance Comparison

```bash
# KEYS is O(N) and blocking - dangerous
time redis-cli KEYS "*"

# SCAN is O(N) total but non-blocking - safe
time redis-cli --scan --count 1000 | wc -l

# The total work is the same, but SCAN yields control between iterations
# allowing other commands to execute
```

## Summary

The `redis-cli --scan` flag is the safe replacement for `KEYS` in production environments. It iterates through keys in configurable batches, allowing other commands to execute between iterations. For deletion, counting, or analysis of key patterns, always use SCAN with `xargs` for batch processing. Remember that SCAN may return duplicates, so deduplicate results when exact counts matter.
