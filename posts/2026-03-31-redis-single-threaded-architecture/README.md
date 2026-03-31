# How Redis Single-Threaded Architecture Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Architecture, Single-Threaded, Performance, Internals

Description: Understand how Redis achieves high throughput with a single-threaded event loop, when multi-threading applies, and the implications for your application design.

---

## The Single-Thread Model

Redis processes client commands on a single main thread. This means all reads and writes to the in-memory data structures happen sequentially, with no locks, no mutex contention, and no context switches between command executions.

The core claim: Redis can handle 100,000+ operations per second on a single core because each operation takes microseconds and the absence of lock overhead eliminates the main bottleneck of multi-threaded systems.

```text
Client 1  ---[SET k1 v1]---->|
Client 2  ---[GET k2    ]---->| Event Loop (single thread)
Client 3  ---[INCR hits ]---->|    executes one at a time
                               |
                               |--> SET k1 v1 --> reply Client 1
                               |--> GET k2    --> reply Client 2
                               |--> INCR hits --> reply Client 3
```

## The Event Loop

Redis uses an event-driven model based on `epoll` (Linux), `kqueue` (macOS/BSD), or `select`. The event loop handles:

1. **Network I/O**: accepting new connections, reading request data
2. **Command execution**: processing the parsed command
3. **Response writing**: sending replies back to clients
4. **Timer events**: running background tasks (key expiration, AOF flush)

```text
Event Loop Iteration:
1. aeApiPoll() - wait for I/O events (epoll_wait)
2. Process file events (read commands from sockets)
3. Execute commands on the data store
4. Write replies back to client buffers
5. Process time events (expiry, stats, replication)
6. Repeat
```

## Why Single-Threading Is Fast

### In-Memory Operations Are CPU-Bound

All data lives in RAM. A simple `GET` involves:
1. Hash the key (O(1))
2. Read from hash table (O(1))
3. Serialize the response

Total: under 1 microsecond. No disk I/O, no network wait inside the operation.

### No Lock Contention

Multi-threaded systems spend significant time on locking. A Redis benchmark shows that lock contention alone can reduce throughput by 30-50% under high write concurrency.

### CPU Cache Efficiency

A single thread working on one data structure at a time is CPU cache-friendly. Multi-threaded access to the same data structures causes cache line invalidation across cores.

## What Happens During a Slow Command

Because commands run sequentially, a slow command blocks all other clients. This is why commands like `KEYS`, `SMEMBERS` on large sets, and `LRANGE 0 -1` on long lists are dangerous in production.

```bash
# This blocks the event loop for hundreds of ms on a 1M key database
KEYS user:*

# Safe alternative - non-blocking cursor scan
SCAN 0 MATCH user:* COUNT 100
```

Use `SLOWLOG GET 10` to find commands exceeding the slowlog threshold:

```bash
redis-cli SLOWLOG GET 10
```

```text
1) 1) (integer) 14        # slowlog entry ID
   2) (integer) 1711900000  # timestamp
   3) (integer) 8500       # duration in microseconds
   4) 1) "KEYS"
      2) "user:*"
```

## Multi-Threading in Redis (Since 6.0)

Redis 6.0 added optional multi-threaded I/O for network reads and writes, while keeping command execution single-threaded:

```text
redis.conf:
io-threads 4
io-threads-do-reads yes
```

With this configuration:
- 4 threads handle reading data from client sockets in parallel
- 4 threads handle writing responses to client sockets in parallel
- The main thread still executes all commands sequentially

This is beneficial when the bottleneck is network I/O rather than command execution (common when handling many large responses).

## Background Threads for Housekeeping

Redis uses background threads for specific non-blocking tasks:

| Thread           | Task                                        |
|------------------|---------------------------------------------|
| BIO_CLOSE_FILE   | Closing file descriptors asynchronously     |
| BIO_AOF_FSYNC    | Flushing AOF to disk                        |
| BIO_LAZY_FREE    | Freeing large memory allocations (UNLINK)   |

The `UNLINK` command deletes a key asynchronously by offloading the actual memory release to a background thread:

```bash
# UNLINK is async - returns immediately, frees memory in background
redis-cli UNLINK large_set

# DEL is sync - blocks until memory is freed
redis-cli DEL large_set
```

## Implications for Application Design

**1. Avoid long-running commands in production:**

```bash
# Bad: blocks the event loop
SORT large_list ALPHA LIMIT 0 1000000

# Good: paginate or pre-compute the sorted view
ZRANGE sorted_cache 0 99
```

**2. Use pipelining to amortize round-trip latency:**

```python
pipe = r.pipeline()
for key in keys:
    pipe.get(key)
results = pipe.execute()
```

**3. Use MULTI/EXEC transactions for atomic batches - no interleaving from other clients:**

```bash
MULTI
INCR counter
EXPIRE counter 3600
EXEC
```

**4. Avoid blocking commands in application threads that also serve user traffic:**

```bash
# BLPOP blocks until an element is available
# Use a dedicated consumer thread/process
BLPOP queue:jobs 30
```

## Summary

Redis achieves exceptional throughput through its single-threaded event loop model: all command executions are sequential, eliminating lock contention and enabling microsecond-level operations. Redis 6.0 added optional multi-threaded network I/O and uses background threads for disk and memory housekeeping, but the core command execution remains single-threaded. Understanding this model is essential for avoiding slow commands that block all clients and for structuring workloads that maximize Redis throughput.
