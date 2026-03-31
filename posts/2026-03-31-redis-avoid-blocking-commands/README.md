# How to Avoid Blocking Commands in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Performance, Blocking, Command, Optimization

Description: Learn which Redis commands block the server and how to replace them with non-blocking alternatives like SCAN, UNLINK, and async operations.

---

Redis is single-threaded for command processing. Any command that takes a long time will block all other clients. Identifying and replacing blocking patterns is essential for consistent performance.

## Commands That Block Redis

| Blocking Command | Problem | Alternative |
|---|---|---|
| `KEYS pattern` | Scans entire keyspace | `SCAN` |
| `DEL large-key` | Synchronous deletion | `UNLINK` |
| `HGETALL big-hash` | Returns huge response | `HSCAN` |
| `SMEMBERS large-set` | Returns entire set | `SSCAN` |
| `SORT` | Complex sort on large dataset | Store sorted data in a `ZSET` |
| `FLUSHDB`/`FLUSHALL` | Deletes all keys synchronously | `FLUSHDB ASYNC` |
| `LRANGE 0 -1` | Returns entire list | Paginate with `LRANGE 0 99` |

## Replace KEYS with SCAN

```bash
# Dangerous: blocks server for entire scan
KEYS user:*

# Safe: iterative, non-blocking
SCAN 0 MATCH "user:*" COUNT 100
```

In Python:

```python
import redis
r = redis.Redis()

# Use scan_iter for automatic cursor management
for key in r.scan_iter("user:*", count=100):
    print(key)
```

## Replace DEL with UNLINK for Large Keys

```bash
# Synchronous deletion - blocks while freeing memory
DEL large-list-key

# Asynchronous deletion - returns immediately, frees memory in background
UNLINK large-list-key
```

## Use HSCAN Instead of HGETALL

```bash
# Blocks if hash has millions of fields
HGETALL big-hash

# Safe iteration
HSCAN big-hash 0 COUNT 100
```

## Use FLUSHDB ASYNC

```bash
# Synchronous - blocks until all keys deleted
FLUSHDB

# Asynchronous - returns immediately
FLUSHDB ASYNC
FLUSHALL ASYNC
```

## Detect Blocking Commands with SLOWLOG

```bash
redis-cli SLOWLOG GET 25
```

Any entry above 1ms (1000 microseconds) in your high-throughput service is worth investigating.

## Avoid Large LRANGE Operations

```bash
# Potentially blocks if list has millions of entries
LRANGE mylist 0 -1

# Paginate instead
LRANGE mylist 0 99      # First 100
LRANGE mylist 100 199   # Next 100
```

## Monitor for Blocking Issues in Real Time

```bash
redis-cli --latency -h 127.0.0.1 -p 6379
```

Spikes in latency correlate with blocking commands executing.

## Summary

Avoid Redis blocking by replacing `KEYS` with `SCAN`, `DEL` with `UNLINK` for large keys, `HGETALL`/`SMEMBERS` with `HSCAN`/`SSCAN` for large collections, and `FLUSHDB` with `FLUSHDB ASYNC`. Use `SLOWLOG` to identify which commands are blocking, and always paginate when reading large data structures.
