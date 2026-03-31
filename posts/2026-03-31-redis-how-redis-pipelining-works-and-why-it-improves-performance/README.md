# How Redis Pipelining Works and Why It Improves Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pipelining, Performance, Network, Throughput

Description: Understand how Redis pipelining batches multiple commands in one network round-trip, reducing RTT overhead and dramatically increasing throughput.

---

## The Network Latency Problem

Without pipelining, each Redis command requires a full round-trip:

1. Client sends command
2. Network transit (e.g., 1ms LAN latency)
3. Redis processes command (microseconds)
4. Network transit back
5. Client receives response

For 1000 commands at 1ms latency each, the total time is dominated by network latency: 1000ms (1 second) even though Redis could process all 1000 commands in under 1ms.

## How Pipelining Works

Pipelining batches multiple commands into a single network write. The client sends all commands without waiting for responses, then reads all responses in one batch.

Without pipelining:
```text
Client      Network      Redis
  |--SET k1---->|          |
  |             |--SET k1->|
  |             |<--OK-----|
  |<---OK-------|          |
  |--SET k2---->|          |
  ... (1 round trip per command)
```

With pipelining:
```text
Client      Network      Redis
  |--SET k1----|          |
  |--SET k2----|          |
  |--SET k3--->|          |
  |            |--batch-->|
  |            |<-OK,OK,OK|
  |<--OK,OK,OK-|          |
  ... (1 round trip for all commands)
```

## Basic Pipeline Example in Python

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379)

# Without pipelining: N round-trips
start = time.time()
for i in range(1000):
    r.set(f'key:{i}', f'value:{i}')
print(f"No pipeline: {time.time() - start:.3f}s")

# With pipelining: 1 round-trip
start = time.time()
pipe = r.pipeline(transaction=False)
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
print(f"With pipeline: {time.time() - start:.3f}s")
```

Typical results on a remote server (1ms RTT):
- Without pipeline: ~1.0s
- With pipeline: ~0.05s (20x improvement)

## Pipeline vs Transaction

By default, `r.pipeline()` uses `MULTI/EXEC` transactions. Use `transaction=False` for pure pipelining without transactional semantics:

```python
# Transactional pipeline (MULTI/EXEC wrapping)
pipe = r.pipeline()          # transaction=True by default
pipe.set('a', 1)
pipe.incr('a')
pipe.execute()               # Wrapped in MULTI/EXEC

# Non-transactional pipeline (no MULTI/EXEC)
pipe = r.pipeline(transaction=False)
pipe.set('a', 1)
pipe.incr('a')
pipe.execute()               # Just batched network writes
```

Non-transactional is faster because it avoids the MULTI/EXEC overhead. Use transactions only when you need atomicity.

## Pipeline in Node.js (ioredis)

```javascript
const Redis = require('ioredis');
const client = new Redis({ host: 'localhost', port: 6379 });

// Create a pipeline
const pipeline = client.pipeline();

for (let i = 0; i < 1000; i++) {
    pipeline.set(`key:${i}`, `value:${i}`);
}

const results = await pipeline.exec();
// results: [[null, 'OK'], [null, 'OK'], ...]
// Each element: [error, result]

console.log(`Set ${results.length} keys`);
```

## Pipeline in Go (go-redis)

```go
package main

import (
    "context"
    "fmt"
    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

    pipe := rdb.Pipeline()

    for i := 0; i < 1000; i++ {
        pipe.Set(ctx, fmt.Sprintf("key:%d", i), fmt.Sprintf("value:%d", i), 0)
    }

    cmds, err := pipe.Exec(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Executed %d commands\n", len(cmds))
}
```

## Reading Results from a Pipeline

All responses are returned together after `execute()`:

```python
pipe = r.pipeline(transaction=False)
pipe.set('name', 'Alice')
pipe.get('name')
pipe.incr('counter')
results = pipe.execute()

# results[0] = True (SET response)
# results[1] = b'Alice' (GET response)
# results[2] = 1 (INCR response)
print(results)
```

## When Pipelining Helps Most

- Many independent SET/GET/DEL operations
- Bulk data loading
- Reporting queries reading many keys
- Remote Redis connections (higher RTT amplifies the benefit)

## When Pipelining Does Not Help

- Commands where each subsequent command depends on the previous result
- Single commands (no batching benefit)
- Commands that are already fast enough without batching

```python
# Cannot pipeline this - each GET result determines next SET
value = r.get('counter')
if int(value) < 100:
    r.set('status', 'low')
```

## Summary

Redis pipelining eliminates the per-command round-trip overhead by sending multiple commands in a single network write and reading all responses together. This provides dramatic throughput improvements (10x-100x) for workloads with many independent commands on high-latency connections. Use `pipeline(transaction=False)` for pure performance; use `pipeline()` (with MULTI/EXEC) only when you need atomic execution of the batch.
