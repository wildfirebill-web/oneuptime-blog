# Redis Key Management Commands Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Key, Command, Cheat Sheet, TTL

Description: Complete Redis key management commands reference covering DEL, EXISTS, EXPIRE, TTL, SCAN, RENAME, COPY, and type inspection.

---

Redis key management commands let you inspect, expire, move, and delete keys. Here is the complete reference for working with keys across all data types.

## Existence and Deletion

```bash
# Check if key exists
EXISTS key1
EXISTS key1 key2 key3   # count how many of these exist

# Delete one or more keys
DEL key1 key2 key3

# Delete key asynchronously (non-blocking, Redis 4.0+)
UNLINK key1 key2    # queues deletion in background
```

## Expiration (TTL)

```bash
# Set TTL in seconds
EXPIRE key 60

# Set TTL in milliseconds
PEXPIRE key 60000

# Set TTL as Unix timestamp (seconds)
EXPIREAT key 1800000000

# Set TTL as Unix timestamp (milliseconds)
PEXPIREAT key 1800000000000

# Conditional expiration (Redis 7.0+)
EXPIRE key 60 NX   # set only if no TTL exists
EXPIRE key 60 XX   # set only if TTL exists
EXPIRE key 60 GT   # set only if new TTL > current TTL
EXPIRE key 60 LT   # set only if new TTL < current TTL

# Remove TTL (make key permanent)
PERSIST key

# Get remaining TTL
TTL key             # seconds (-1 = no TTL, -2 = key not found)
PTTL key            # milliseconds
```

## Key Type and Encoding

```bash
# Get data type
TYPE key    # returns string, list, set, zset, hash, stream

# Get internal encoding
OBJECT ENCODING key   # e.g., listpack, quicklist, ziplist, hashtable

# Get LRU idle time (seconds since last access)
OBJECT IDLETIME key

# Get LFU access frequency
OBJECT FREQ key

# Get reference count (usually 1)
OBJECT REFCOUNT key

# Get approximate memory usage (bytes)
MEMORY USAGE key
MEMORY USAGE key SAMPLES 5   # sampling depth for nested structures
```

## Key Discovery

```bash
# WARNING: KEYS blocks the server. Use SCAN in production.
KEYS "*"
KEYS "user:*"
KEYS "session:????"    # ? matches one character

# Cursor-based scan (safe for production)
SCAN 0 COUNT 100
SCAN 0 MATCH "user:*" COUNT 100
SCAN 0 MATCH "*" TYPE hash     # filter by data type
SCAN 0 MATCH "session:*" COUNT 100 TYPE string
```

## Renaming and Moving

```bash
# Rename a key
RENAME oldkey newkey          # overwrites newkey if it exists
RENAMENX oldkey newkey        # only rename if newkey doesn't exist

# Move key to another database
MOVE key 1       # move to database 1

# Copy key to another key (Redis 6.2+)
COPY source destination
COPY source destination DB 2          # copy to different database
COPY source destination REPLACE       # overwrite if dest exists
```

## Random Key

```bash
# Return a random key from the current database
RANDOMKEY
```

## Key Dump and Restore

```bash
# Serialize key value (for RESTORE)
DUMP key

# Restore serialized value
RESTORE key 0 <serialized_value>           # 0 = no TTL
RESTORE key 60000 <serialized_value>       # TTL in milliseconds
RESTORE key 0 <serialized_value> REPLACE   # overwrite if exists
```

## Summary

Redis key management commands cover the full lifecycle of a key: creation via data type commands, TTL management with EXPIRE and PERSIST, type inspection with TYPE and OBJECT ENCODING, and safe iteration with SCAN. Always use SCAN instead of KEYS in production to avoid blocking the event loop.
