# How to Use KEYS Pattern Matching in Redis (and Why to Avoid It)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Key Management, Performance, Best Practices, Commands

Description: Learn how to use the KEYS command with glob patterns in Redis, understand why it blocks production servers, and discover safe alternatives like SCAN.

---

## What Is KEYS in Redis

`KEYS` returns all keys in the current database that match a glob-style pattern. It is the most direct way to search for keys by name - but it comes with a critical caveat that makes it unsuitable for production use.

```text
KEYS pattern
```

Returns an array of all matching keys.

## Glob Pattern Syntax

| Pattern | Matches |
|---------|---------|
| `*` | Any string (including empty) |
| `?` | Any single character |
| `[abc]` | Any of a, b, or c |
| `[a-z]` | Any character in range |
| `[^abc]` | Any character NOT in set |
| `\*` | Literal asterisk |

## Basic Usage

```bash
# All keys
KEYS *

# Keys starting with "user:"
KEYS user:*

# Keys with exactly one character after "key:"
KEYS key:?

# Specific patterns
KEYS session:[0-9]*
KEYS user:*:profile

# Exact key (no wildcards)
KEYS myexactkey
```

## Why KEYS Blocks Production Servers

`KEYS` is an O(N) operation where N is the total number of keys in the database. During execution, **Redis is single-threaded and cannot process any other commands**. On a database with millions of keys, this can take seconds and cause:

- Timeouts for all other clients
- Cascading failures in downstream services
- Apparent "Redis is down" situations
- SLA violations

```bash
# On a database with 10 million keys, this could block for seconds:
KEYS *  # NEVER do this in production!
```

## When KEYS Is Acceptable

- Development and debugging on small datasets
- In scripts run during maintenance windows with no traffic
- On dedicated admin connections with replicated databases
- Testing with tiny datasets

## The Right Alternative: SCAN

Use `SCAN` in production for all key iteration needs:

```bash
# Instead of: KEYS user:*
# Use cursor-based iteration:
SCAN 0 MATCH user:* COUNT 100
# Repeat with returned cursor until cursor = 0
```

## Practical Example in Python

```python
import redis
import time

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# NEVER in production - shown for educational purposes only
def bad_pattern_search(pattern):
    """DO NOT use in production with large datasets."""
    print("WARNING: KEYS blocks the server!")
    start = time.perf_counter()
    keys = client.keys(pattern)
    elapsed = (time.perf_counter() - start) * 1000
    print(f"KEYS completed in {elapsed:.2f}ms, found {len(keys)} keys")
    return keys

# Production-safe alternative
def good_pattern_search(pattern, count=100):
    """Safe SCAN-based key search."""
    start = time.perf_counter()
    keys = list(client.scan_iter(match=pattern, count=count))
    elapsed = (time.perf_counter() - start) * 1000
    print(f"SCAN completed in {elapsed:.2f}ms, found {len(keys)} keys")
    return keys

# For debugging on small local dev databases:
if False:  # Set to True only in development
    user_keys = bad_pattern_search('user:*')

# Always safe:
user_keys = good_pattern_search('user:*')
```

## Practical Example in Node.js

```javascript
const { createClient } = require('redis');

const client = createClient();
await client.connect();

// NOT for production - educational demonstration only
async function badSearch(pattern) {
  console.warn('WARNING: KEYS blocks the Redis server!');
  return await client.keys(pattern);
}

// Production-safe version using scan
async function safeSearch(pattern) {
  const keys = [];
  for await (const key of client.scanIterator({ MATCH: pattern, COUNT: 100 })) {
    keys.push(key);
  }
  return keys;
}

// Always use the safe version
const userKeys = await safeSearch('user:*');
console.log(`Found ${userKeys.length} user keys`);
```

## Common Pattern Use Cases

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Find all sessions (safe version with SCAN)
def find_sessions():
    return list(client.scan_iter(match='session:*'))

# Find keys for a specific user
def find_user_keys(user_id):
    return list(client.scan_iter(match=f'user:{user_id}:*'))

# Find all cache keys
def find_cache_keys():
    return list(client.scan_iter(match='cache:*'))

# Find keys with date suffix pattern
def find_daily_keys(date_prefix='2026-03'):
    return list(client.scan_iter(match=f'report:{date_prefix}-*'))
```

## Disabling KEYS in Production

Prevent accidental `KEYS` usage with ACL rules or command renaming:

```bash
# Rename KEYS to something harder to accidentally use
# In redis.conf:
rename-command KEYS ""  # Completely disable

# Or restrict to admin users only
ACL SETUSER app_user on >pass ~* &* +@all -keys
ACL SETUSER admin_user on >adminpass ~* &* +@all
```

## Diagnosing Accidental KEYS Usage

Check if `KEYS` is being called via the slowlog:

```bash
CONFIG SET slowlog-log-slower-than 0  # Log everything

SLOWLOG GET 10
# Shows recent slow commands - look for 'keys' entries
```

Or via MONITOR (development only):

```bash
redis-cli MONITOR | grep -i "^.*\"keys\""
```

## KEYS on Replica/Slave Connections

One acceptable use of `KEYS` in production is on a read-only replica that receives no production traffic:

```python
import redis

# Connect to replica for admin queries
replica_client = redis.Redis(
    host='redis-replica.internal',
    port=6379,
    decode_responses=True
)

# KEYS is less harmful on a dedicated replica
admin_keys = replica_client.keys('admin:*')
print(f"Admin keys: {admin_keys}")
```

This still blocks the replica during the call but does not affect the primary or its clients.

## Summary

`KEYS` returns all matching keys using glob patterns but is an O(N) blocking operation that freezes the Redis server during execution. Use it only in development or on non-production replicas. In production, always replace `KEYS` with cursor-based `SCAN` iteration, which processes keys in batches allowing other clients to continue operating. Consider disabling `KEYS` entirely in production with ACL rules or command renaming.
