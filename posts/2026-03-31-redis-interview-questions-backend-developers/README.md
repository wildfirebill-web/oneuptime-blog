# Top Redis Interview Questions for Backend Developers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Interview, Backend, Cache, Data Structure

Description: Essential Redis interview questions for backend developers, covering caching strategies, data structures, persistence, and common pitfalls.

---

If you are interviewing for a backend developer role, Redis knowledge is often expected. This post covers the most common Redis interview questions with concise, accurate answers.

## Core Concept Questions

**Q: What is Redis and what makes it fast?**

Redis is an in-memory data store. Speed comes from keeping all data in RAM, using non-blocking I/O with an event loop, and simple O(1) or O(log N) data structure operations.

**Q: What data types does Redis support?**

- String - binary-safe, up to 512 MB
- Hash - field-value pairs
- List - ordered sequences (linked list / quicklist)
- Set - unordered unique values
- Sorted Set - unique values ranked by score
- Stream - append-only log
- HyperLogLog, Bitmap, Geospatial indexes

**Q: What is the difference between EXPIRE and PERSIST?**

```bash
EXPIRE key 60   # set TTL to 60 seconds
PERSIST key     # remove TTL, make key permanent
TTL key         # returns remaining time in seconds
```

## Caching Questions

**Q: What cache eviction policies does Redis support?**

Common policies set via `maxmemory-policy`:

- `noeviction` - return errors when memory is full
- `allkeys-lru` - evict least recently used keys
- `volatile-lru` - LRU only among keys with TTL
- `allkeys-lfu` - evict least frequently used keys

**Q: Explain cache-aside vs write-through caching.**

Cache-aside: application checks cache first, then loads from DB on miss and writes to cache. Write-through: every write goes to both cache and DB simultaneously.

## Persistence Questions

**Q: What is the difference between RDB and AOF?**

RDB creates point-in-time snapshots. AOF logs every write operation. RDB is faster to restore but can lose more data on crash. AOF offers better durability at the cost of larger files and slower write performance.

**Q: How does Redis handle data loss on restart?**

```bash
# Check persistence config
redis-cli CONFIG GET save
redis-cli CONFIG GET appendonly
```

With neither enabled, all data is lost on restart. With RDB, you lose data since the last snapshot. With AOF, you lose at most one second of writes (with `appendfsync everysec`).

## Practical Questions

**Q: How do you handle the thundering herd / cache stampede problem?**

Use probabilistic early expiration or a mutex lock pattern. When a key is about to expire, only one client refreshes it while others serve stale data:

```python
import redis
import time

r = redis.Redis()
LOCK_KEY = "lock:product:42"
DATA_KEY = "product:42"

# Try to acquire lock before key expires
if r.set(LOCK_KEY, "1", nx=True, ex=5):
    data = fetch_from_db(42)
    r.set(DATA_KEY, data, ex=300)
    r.delete(LOCK_KEY)
```

**Q: What is Redis pipelining?**

Pipelining sends multiple commands to Redis without waiting for individual responses, reducing round-trip latency:

```python
pipe = r.pipeline()
pipe.set("a", 1)
pipe.set("b", 2)
pipe.get("a")
pipe.execute()  # all three commands sent at once
```

## Summary

Backend Redis interview questions focus on data structures, caching strategies, persistence options, and performance patterns like pipelining and cache stampede prevention. Understanding when to use each eviction policy and persistence mode demonstrates production-level knowledge that interviewers look for.
