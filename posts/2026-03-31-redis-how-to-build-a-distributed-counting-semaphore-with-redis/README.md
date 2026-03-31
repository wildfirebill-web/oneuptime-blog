# How to Build a Distributed Counting Semaphore with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Semaphore, Distributed Locking, Concurrency, Lua Script

Description: Build a distributed counting semaphore with Redis to limit concurrent access to a shared resource across multiple application servers.

---

## Overview

A counting semaphore limits the number of concurrent actors accessing a shared resource. Unlike a mutex (which allows only one), a semaphore allows N concurrent holders. In distributed systems, this is critical for controlling database connection pool sizes, limiting concurrent API calls to third-party services, or throttling background job workers.

## Semaphore Design

The semaphore uses a Redis Sorted Set to track holders by acquisition timestamp, enabling automatic expiration of stale holders:

```text
semaphore:{name}  -> Sorted Set: {holder_id: timestamp}
Max size: N (the limit)
Holder TTL: configurable (to recover from crashed holders)
```

## Acquiring the Semaphore

```python
import time
import uuid
from redis import Redis

r = Redis(host='localhost', port=6379, decode_responses=True)

ACQUIRE_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local holder_id = ARGV[2]
local timestamp = tonumber(ARGV[3])
local timeout = tonumber(ARGV[4])

-- Remove expired holders
redis.call('ZREMRANGEBYSCORE', key, 0, timestamp - timeout)

-- Check if there is room
local count = redis.call('ZCARD', key)
if count < limit then
    redis.call('ZADD', key, timestamp, holder_id)
    return 1
end
return 0
"""

RELEASE_SCRIPT = """
redis.call('ZREM', KEYS[1], ARGV[1])
return 1
"""

REFRESH_SCRIPT = """
local exists = redis.call('ZSCORE', KEYS[1], ARGV[1])
if exists then
    redis.call('ZADD', KEYS[1], tonumber(ARGV[2]), ARGV[1])
    return 1
end
return 0
"""

class DistributedSemaphore:
    def __init__(
        self,
        name: str,
        limit: int,
        holder_timeout: float = 30.0,
        retry_interval: float = 0.1,
        max_wait: float = 10.0
    ):
        self.key = f"semaphore:{name}"
        self.limit = limit
        self.holder_timeout = holder_timeout
        self.retry_interval = retry_interval
        self.max_wait = max_wait

        self._acquire_fn = r.register_script(ACQUIRE_SCRIPT)
        self._release_fn = r.register_script(RELEASE_SCRIPT)
        self._refresh_fn = r.register_script(REFRESH_SCRIPT)

    def acquire(self, holder_id: str = None) -> str | None:
        """Try to acquire the semaphore. Returns holder_id or None."""
        holder_id = holder_id or str(uuid.uuid4())
        deadline = time.time() + self.max_wait

        while time.time() < deadline:
            now = time.time()
            acquired = self._acquire_fn(
                keys=[self.key],
                args=[self.limit, holder_id, now, self.holder_timeout]
            )
            if acquired:
                return holder_id
            time.sleep(self.retry_interval)

        return None

    def release(self, holder_id: str) -> bool:
        """Release the semaphore."""
        return bool(self._release_fn(keys=[self.key], args=[holder_id]))

    def refresh(self, holder_id: str) -> bool:
        """Refresh holder timeout to prevent expiration."""
        now = time.time()
        return bool(self._refresh_fn(keys=[self.key], args=[holder_id, now]))

    def available_count(self) -> int:
        """Get number of available slots."""
        now = time.time()
        # Count active (non-expired) holders
        r.zremrangebyscore(self.key, 0, now - self.holder_timeout)
        held = r.zcard(self.key)
        return max(0, self.limit - held)

    def holder_count(self) -> int:
        """Get current holder count."""
        now = time.time()
        r.zremrangebyscore(self.key, 0, now - self.holder_timeout)
        return r.zcard(self.key)
```

## Context Manager Usage

```python
from contextlib import contextmanager

@contextmanager
def semaphore(name: str, limit: int, timeout: float = 30.0):
    """Context manager for automatic semaphore release."""
    sem = DistributedSemaphore(name, limit, holder_timeout=timeout)
    holder_id = sem.acquire()
    if holder_id is None:
        raise TimeoutError(f"Could not acquire semaphore '{name}' within timeout")
    try:
        yield holder_id
    finally:
        sem.release(holder_id)

# Usage
def call_external_api(data: dict) -> dict:
    """Limit to 5 concurrent calls to the external API."""
    with semaphore("external_api", limit=5, timeout=30.0):
        response = make_api_call(data)
        return response
```

## Database Connection Pool Limiter

```python
def query_database_with_limit(query: str, params: tuple) -> list:
    """Limit concurrent DB connections across all app instances."""
    sem = DistributedSemaphore("db_connections", limit=20)
    holder_id = sem.acquire()
    if not holder_id:
        raise Exception("Database connection pool exhausted")

    try:
        return execute_db_query(query, params)
    finally:
        sem.release(holder_id)
```

## Background Job Worker Throttle

```python
import threading

def worker_with_semaphore(worker_id: int, sem: DistributedSemaphore):
    """Worker that respects the semaphore limit."""
    holder_id = sem.acquire()
    if not holder_id:
        print(f"Worker {worker_id}: Could not acquire slot, skipping")
        return

    try:
        print(f"Worker {worker_id}: Running job")
        # Simulate work
        time.sleep(2)
    finally:
        sem.release(holder_id)
        print(f"Worker {worker_id}: Released slot")

# Start 10 workers but limit to 3 concurrent
sem = DistributedSemaphore("job_workers", limit=3, max_wait=5.0)
threads = [
    threading.Thread(target=worker_with_semaphore, args=(i, sem))
    for i in range(10)
]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

## Monitoring the Semaphore

```python
def get_semaphore_holders(name: str) -> list[dict]:
    """Get current holders with their acquisition timestamps."""
    key = f"semaphore:{name}"
    entries = r.zrange(key, 0, -1, withscores=True)
    now = time.time()
    return [
        {
            "holder_id": hid,
            "acquired_at": ts,
            "held_for_seconds": round(now - ts, 1)
        }
        for hid, ts in entries
    ]
```

## Summary

A Redis counting semaphore uses a Sorted Set with acquisition timestamps to track concurrent holders. Atomic Lua scripts ensure the limit is respected even under concurrent acquisition attempts from multiple servers. Automatic expiration of holders with old timestamps prevents semaphore starvation when processes crash without releasing, making the semaphore self-healing in production environments.
