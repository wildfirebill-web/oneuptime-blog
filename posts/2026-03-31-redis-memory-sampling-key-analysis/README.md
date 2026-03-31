# How to Use Redis Memory Sampling for Key Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Memory, Sampling, Analysis

Description: Analyze Redis memory distribution using MEMORY USAGE sampling, SCAN-based analysis, and redis-cli --bigkeys to find the largest and most expensive keys.

---

You rarely know exactly which keys are consuming the most memory in production. Redis provides several sampling tools to identify hot spots without scanning every key.

## redis-cli --bigkeys

The fastest way to find large keys is the built-in `--bigkeys` flag:

```bash
redis-cli --bigkeys
```

Output:

```text
Scanning the entire keyspace to find biggest keys as well as
average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
per 100 SCAN calls to avoid hitting your Redis instance too hard.

-------- summary -------
Biggest string found "user:profile:99" has 12345 bytes
Biggest list found "notifications:44" has 4891 items
Biggest hash found "product:catalog:22" has 892 fields
...
```

Add `-i 0.1` to slow the scan and reduce impact on production:

```bash
redis-cli -i 0.1 --bigkeys
```

## MEMORY USAGE Sampling via SCAN

For more control, combine `SCAN` with `MEMORY USAGE`:

```python
import redis
import heapq

r = redis.Redis(decode_responses=True)

def sample_top_keys(r, pattern="*", sample_limit=10000, top_n=20):
    """Scan up to sample_limit keys and return top_n by memory."""
    top = []  # min-heap of (bytes, key)
    cursor = 0
    sampled = 0

    while sampled < sample_limit:
        cursor, keys = r.scan(cursor, match=pattern, count=200)
        for key in keys:
            mem = r.memory_usage(key, samples=1)
            if mem is None:
                continue
            if len(top) < top_n:
                heapq.heappush(top, (mem, key))
            elif mem > top[0][0]:
                heapq.heapreplace(top, (mem, key))
            sampled += 1
        if cursor == 0 or sampled >= sample_limit:
            break

    return sorted(top, reverse=True)

top_keys = sample_top_keys(r, pattern="*", sample_limit=5000, top_n=20)
for mem, key in top_keys:
    ktype = r.type(key)
    print(f"{mem:8d} bytes  [{ktype:10s}]  {key}")
```

## MEMORY DOCTOR

Redis provides a built-in diagnostic:

```bash
redis-cli MEMORY DOCTOR
```

Possible outputs:

```text
"Hi Sam, I have a few issues with this instance that you should know about:
 * High RSS memory usage (~2.15x used memory). This usually means there was
   a memory fragmentation issue in the past."

"Sam, I can't find any memory issues with this instance."
```

## Analyzing Memory by Key Prefix

Group memory usage by key namespace:

```python
from collections import defaultdict

def memory_by_prefix(r, delimiter=":", max_keys=50000):
    prefix_memory = defaultdict(int)
    prefix_count = defaultdict(int)
    cursor = 0
    processed = 0

    while processed < max_keys:
        cursor, keys = r.scan(cursor, count=500)
        for key in keys:
            prefix = key.split(delimiter)[0] if delimiter in key else key
            mem = r.memory_usage(key, samples=0) or 0
            prefix_memory[prefix] += mem
            prefix_count[prefix] += 1
            processed += 1
        if cursor == 0:
            break

    results = [
        {
            "prefix": p,
            "total_mb": prefix_memory[p] / 1024 / 1024,
            "key_count": prefix_count[p],
            "avg_bytes": prefix_memory[p] / prefix_count[p],
        }
        for p in prefix_memory
    ]
    return sorted(results, key=lambda x: x["total_mb"], reverse=True)

for row in memory_by_prefix(r)[:10]:
    print(
        f"{row['prefix']:20s}  "
        f"{row['total_mb']:.1f} MB  "
        f"{row['key_count']:6d} keys  "
        f"{row['avg_bytes']:.0f} avg bytes"
    )
```

## INFO MEMORY Snapshot

For a high-level overview without scanning:

```bash
redis-cli INFO memory
```

Key fields:

```text
used_memory:          102400000   # heap memory in use
used_memory_rss:      134217728   # RSS reported by OS
mem_fragmentation_ratio: 1.31     # RSS / used_memory
used_memory_peak:     115000000   # peak heap usage
```

## MEMORY STATS Deep Dive

```bash
redis-cli MEMORY STATS
```

This breaks down memory into categories:

```text
peak.allocated       = 115 MB  (peak heap)
total.allocated      = 102 MB  (current heap)
startup.allocated    = 800 KB  (server baseline)
replication.backlog  = 1 MB
clients.slaves       = 20 KB
clients.normal       = 50 KB
aof.buffer           = 0
keys.count           = 500000
keys.bytes-per-key   = 204     (average key cost)
```

## Sampling Without Impacting Production

Full `MEMORY USAGE` with `samples=0` (exact) is expensive for large collections. Use `samples=1` or `samples=5` for speed:

```bash
# samples=0 means traverse all nested objects (slow for big hashes)
MEMORY USAGE myhash SAMPLES 0

# samples=5 means sample 5 elements to estimate (fast)
MEMORY USAGE myhash SAMPLES 5
```

## Summary

Redis memory sampling starts with `redis-cli --bigkeys` for a fast overview. For production analysis without disruption, combine `SCAN` + `MEMORY USAGE` with `samples=1` to sample a representative subset. Group results by key prefix to identify which namespaces are consuming the most memory, and use `MEMORY STATS` for server-wide breakdown without scanning individual keys.
