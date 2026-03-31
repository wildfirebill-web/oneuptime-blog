# How to Use SCAN in Redis to Iterate Over Keys Safely

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Key Management, Performance, Command, Best Practice

Description: Learn how to use SCAN in Redis to safely iterate over keys using a cursor-based approach that avoids blocking the server, unlike the dangerous KEYS command.

---

## What Is SCAN in Redis

`SCAN` provides a cursor-based iteration over the keyspace. Unlike `KEYS` which blocks the server until it returns all matching keys, `SCAN` returns a small batch of keys per call, allowing other clients to continue operating between iterations.

```text
SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]
```

- `cursor` - start with `0`, continue with the cursor returned by each call until it returns `0` again
- `MATCH` - filter results by glob pattern
- `COUNT` - hint for how many elements to return per call (not guaranteed)
- `TYPE` - filter by data type (Redis 6.0+)

## Basic Usage

```bash
# Start scanning from cursor 0
SCAN 0
# 1) "6821"    <- next cursor to use
# 2) 1) "key:100"
#    2) "user:1"
#    3) "session:abc"

# Continue with returned cursor
SCAN 6821
# 1) "3210"
# 2) 1) "key:200"
#    ...

# When cursor returns 0, iteration is complete
SCAN 3210
# 1) "0"      <- done!
# 2) 1) "key:300"
```

## Complete Iteration Pattern

```bash
# Keep scanning until cursor returns to 0
SCAN 0 MATCH user:* COUNT 100
# ... repeat with returned cursor until you get 0
```

## Practical Example in Python

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Iterate all keys with a pattern
def scan_keys(pattern='*', count=100, key_type=None):
    """Safely iterate over all matching keys."""
    all_keys = []
    cursor = 0

    while True:
        kwargs = {'match': pattern, 'count': count}
        if key_type:
            kwargs['_type'] = key_type

        cursor, keys = client.scan(cursor, **kwargs)
        all_keys.extend(keys)

        if cursor == 0:
            break

    return all_keys

# Find all user keys
user_keys = scan_keys(pattern='user:*')
print(f"Found {len(user_keys)} user keys")

# Find all hash keys
hash_keys = scan_keys(key_type='hash')
print(f"Found {len(hash_keys)} hash keys")

# Use redis-py's built-in scan_iter for convenience
for key in client.scan_iter(match='session:*', count=100):
    print(f"Session key: {key}")
```

## Practical Example in Node.js

```javascript
const { createClient } = require('redis');

const client = createClient();
await client.connect();

async function scanAllKeys(pattern = '*', count = 100) {
  const keys = [];
  let cursor = 0;

  do {
    const result = await client.scan(cursor, {
      MATCH: pattern,
      COUNT: count
    });
    cursor = result.cursor;
    keys.push(...result.keys);
  } while (cursor !== 0);

  return keys;
}

const userKeys = await scanAllKeys('user:*');
console.log(`Found ${userKeys.length} user keys`);

// Using async iterator
for await (const key of client.scanIterator({ MATCH: 'cache:*', COUNT: 50 })) {
  console.log(`Cache key: ${key}`);
}
```

## Practical Example in Go

```go
package main

import (
    "context"
    "fmt"
    "github.com/redis/go-redis/v9"
)

func scanKeys(ctx context.Context, rdb *redis.Client, pattern string) ([]string, error) {
    var keys []string
    iter := rdb.Scan(ctx, 0, pattern, 100).Iterator()

    for iter.Next(ctx) {
        keys = append(keys, iter.Val())
    }

    return keys, iter.Err()
}

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

    keys, err := scanKeys(ctx, rdb, "user:*")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Found %d user keys\n", len(keys))
}
```

## Filtering by Type with SCAN

```bash
# Find only hash keys matching a pattern (Redis 6.0+)
SCAN 0 MATCH user:* TYPE hash COUNT 100
```

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def scan_by_type(pattern='*', key_type='hash', count=100):
    """Get all keys of a specific type."""
    result = []
    cursor = 0
    while True:
        cursor, keys = client.scan(cursor, match=pattern, count=count, _type=key_type)
        result.extend(keys)
        if cursor == 0:
            break
    return result

hash_users = scan_by_type(pattern='user:*', key_type='hash')
list_queues = scan_by_type(pattern='queue:*', key_type='list')
```

## Bulk Delete by Pattern

Use `SCAN` with `DEL` or `UNLINK` for safe bulk deletion:

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def delete_by_pattern(pattern, batch_size=100):
    """Delete all keys matching pattern using SCAN + UNLINK."""
    deleted = 0
    cursor = 0

    while True:
        cursor, keys = client.scan(cursor, match=pattern, count=batch_size)
        if keys:
            # UNLINK is async and non-blocking
            client.unlink(*keys)
            deleted += len(keys)
            print(f"Deleted batch of {len(keys)} keys")

        if cursor == 0:
            break

    print(f"Total deleted: {deleted}")
    return deleted

# Delete all cache keys
delete_by_pattern('cache:*')
```

## COUNT Is Just a Hint

The `COUNT` parameter suggests how many elements Redis should return per call, but the actual count may vary. For small keyspaces, `SCAN` might return all keys in a single call even with `COUNT 1`:

```bash
# With only 10 keys total, may return all in one call
SCAN 0 COUNT 5
# 1) "0"  <- cursor immediately at 0
# 2) 1) ... all 10 keys ...
```

## SCAN Guarantees

- Full iteration: every key present for the entire iteration will be returned
- May return duplicates: code should handle this
- May miss keys that were added/deleted during iteration
- Safe for production: does not block other clients

## HSCAN, SSCAN, ZSCAN for Data Structures

Similar cursor-based iteration is available within data structures:

```bash
# Scan hash fields
HSCAN myhash 0 MATCH field:* COUNT 50

# Scan set members
SSCAN myset 0 MATCH prefix:* COUNT 50

# Scan sorted set members
ZSCAN myzset 0 MATCH member:* COUNT 50
```

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Iterate all fields in a large hash
def scan_hash(key, pattern='*', count=100):
    all_fields = {}
    cursor = 0
    while True:
        cursor, fields = client.hscan(key, cursor, match=pattern, count=count)
        all_fields.update(fields)
        if cursor == 0:
            break
    return all_fields

client.hset('large:config', mapping={f'setting:{i}': f'value:{i}' for i in range(1000)})
config = scan_hash('large:config', pattern='setting:1*')
print(f"Found {len(config)} matching settings")
```

## Summary

`SCAN` is the production-safe way to iterate over Redis keys, using a cursor-based approach that yields small batches without blocking the server. Always use `SCAN` instead of `KEYS` in production. Use `MATCH` to filter by pattern, `TYPE` to filter by data type, and handle potential duplicates in your code. For iterating within data structures, use `HSCAN`, `SSCAN`, and `ZSCAN`.
