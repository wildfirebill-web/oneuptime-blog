# How to Use Redis CLI --scan for Safe Key Enumeration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis CLI, Key Enumeration, Scan, Production Safety

Description: Learn how to use the Redis CLI --scan flag to safely enumerate keys in production without blocking the server, unlike the dangerous KEYS command.

---

## Why Not Use KEYS in Production

The `KEYS` command scans the entire keyspace in one shot and blocks the Redis event loop while doing so. On a large dataset this can cause latency spikes of hundreds of milliseconds or even seconds, making your application temporarily unresponsive.

```bash
# DANGEROUS on large keyspaces - blocks the server
redis-cli KEYS "user:*"
```

The `--scan` flag uses the `SCAN` command internally, which iterates the keyspace in small increments using a cursor, never blocking for long.

## Using --scan for Key Enumeration

The basic `--scan` flag returns all keys in the keyspace:

```bash
redis-cli --scan
```

This outputs one key per line and is safe to run in production:

```text
user:1001
session:abc123
cache:product:42
user:1002
```

## Filtering with --pattern

Combine `--scan` with `--pattern` to filter keys by glob:

```bash
redis-cli --scan --pattern "user:*"
```

```text
user:1001
user:1002
user:1003
```

Common glob patterns:

```text
user:*          - all keys starting with user:
*:session:*     - keys containing :session:
cache:product:? - single-character suffix
order:[0-9]*    - keys starting with a digit after order:
```

## Controlling Scan Count

The `--count` option is a hint to Redis about how many keys to return per iteration. It does not guarantee an exact number but adjusts the work done per cursor step:

```bash
redis-cli --scan --pattern "session:*" --count 200
```

Higher count values reduce the number of round trips but increase the work done per iteration. A value between 100 and 500 is generally safe for production.

## Piping --scan Output to Other Commands

Because `--scan` outputs one key per line, you can pipe it to `xargs` for bulk operations.

**Delete all keys matching a pattern:**

```bash
redis-cli --scan --pattern "cache:temp:*" | xargs redis-cli DEL
```

**Set TTL on keys missing expiry:**

```bash
redis-cli --scan --pattern "session:*" | xargs -I{} redis-cli EXPIRE {} 3600
```

**Count matching keys:**

```bash
redis-cli --scan --pattern "user:*" | wc -l
```

**Export key names to a file:**

```bash
redis-cli --scan --pattern "product:*" > product_keys.txt
```

## Using --scan with a Specific Database

By default `--scan` operates on database 0. Use `-n` to target a different database:

```bash
redis-cli -n 3 --scan --pattern "tmp:*"
```

## Scanning in a Shell Script

Here is a production-safe script that deletes expired temporary keys in batches:

```bash
#!/bin/bash

PATTERN="tmp:*"
BATCH_SIZE=100

redis-cli --scan --pattern "$PATTERN" --count $BATCH_SIZE | \
  while IFS= read -r key; do
    redis-cli DEL "$key" > /dev/null
  done

echo "Cleanup complete"
```

## Scanning with Authentication and TLS

```bash
redis-cli -h redis.example.com -p 6379 -a "$REDIS_PASSWORD" \
  --tls --cacert /etc/ssl/redis-ca.crt \
  --scan --pattern "user:*"
```

## SCAN vs --scan: Understanding the Difference

The `--scan` flag in the CLI is a convenience wrapper around the `SCAN` command. When you call `--scan`, the CLI manages the cursor loop for you automatically:

```bash
# Manual SCAN with cursor management
redis-cli SCAN 0 MATCH "user:*" COUNT 100
# Returns: next_cursor + partial results
# You must call again with returned cursor until cursor == 0

# --scan handles cursor management automatically
redis-cli --scan --pattern "user:*"
# Returns: all matching keys, one per line
```

## Scanning from Application Code

If you need to enumerate keys from application code, replicate the `--scan` behavior using the SCAN command in a loop:

```python
import redis

r = redis.Redis(host='localhost', port=6379)
cursor = 0

while True:
    cursor, keys = r.scan(cursor=cursor, match='user:*', count=200)
    for key in keys:
        print(key.decode())
    if cursor == 0:
        break
```

```javascript
const redis = require('redis');
const client = redis.createClient();

async function scanKeys(pattern) {
  let cursor = 0;
  const results = [];
  do {
    const reply = await client.scan(cursor, { MATCH: pattern, COUNT: 200 });
    cursor = reply.cursor;
    results.push(...reply.keys);
  } while (cursor !== 0);
  return results;
}
```

## Summary

The `--scan` flag in Redis CLI is the production-safe way to enumerate keys. It uses the cursor-based `SCAN` command under the hood, avoiding the blocking behavior of `KEYS`. Combined with `--pattern` for filtering and piped to `xargs` for bulk operations, it is a powerful tool for key inspection, cleanup, and auditing in live Redis deployments.
