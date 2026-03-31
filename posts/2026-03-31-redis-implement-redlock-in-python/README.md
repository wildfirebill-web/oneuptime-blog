# How to Implement Redlock in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redlock, Distributed Locks, Python, Concurrency, Mutex

Description: Implement Redlock distributed locking in Python using the redis-py built-in lock or the redlock-py library across multiple Redis nodes for fault-tolerant mutual exclusion.

---

## What is Redlock

Redlock is a distributed locking algorithm designed by Redis creator Salvatore Sanfilippo. It acquires a lock on N independent Redis nodes (typically 5) and considers the lock acquired if a majority (N/2 + 1) of nodes grant it. This provides safety even if some Redis nodes fail.

## Single-Node Lock with redis-py

For most applications, a single-node lock using `redis-py` built-in lock is sufficient:

```python
import redis
import time

r = redis.Redis(decode_responses=True)

def process_with_lock():
    lock = r.lock("resource:critical_section", timeout=10, blocking_timeout=5)
    try:
        acquired = lock.acquire()
        if not acquired:
            print("Could not acquire lock - another process has it")
            return
        print("Lock acquired - processing...")
        time.sleep(2)
        print("Done processing")
    finally:
        try:
            lock.release()
            print("Lock released")
        except redis.exceptions.LockNotOwnedError:
            print("Lock expired before release")

process_with_lock()
```

## Context Manager Usage

```python
import redis

r = redis.Redis()

def safe_update(resource_id: str):
    with r.lock(f"lock:{resource_id}", timeout=30, blocking_timeout=10) as lock:
        # Critical section - guaranteed exclusive access
        current = r.get(resource_id)
        new_value = int(current or 0) + 1
        r.set(resource_id, new_value)
        return new_value

result = safe_update("counter:orders")
print(f"Updated value: {result}")
```

## Multi-Node Redlock with redlock-py

Install the library:

```bash
pip install redlock-py
```

```python
from redlock import Redlock, RedLockError
import redis

# Create connections to 3 independent Redis nodes
redis_instances = [
    redis.StrictRedis(host="redis-node-1", port=6379),
    redis.StrictRedis(host="redis-node-2", port=6379),
    redis.StrictRedis(host="redis-node-3", port=6379),
]

dlm = Redlock(redis_instances, retry_count=3, retry_delay=200)

def process_with_redlock(resource: str):
    try:
        with dlm.lock(resource, 10000) as lock:
            print(f"Lock acquired on {resource}")
            # Critical section
            time.sleep(2)
            print(f"Work done on {resource}")
    except RedLockError:
        print("Could not acquire lock after retries")

process_with_redlock("shared:payment:order:5001")
```

## Manual Redlock Implementation

For educational purposes, here is the core algorithm:

```python
import redis
import time
import uuid

nodes = [
    redis.StrictRedis(host="redis-1", port=6379),
    redis.StrictRedis(host="redis-2", port=6379),
    redis.StrictRedis(host="redis-3", port=6379),
    redis.StrictRedis(host="redis-4", port=6379),
    redis.StrictRedis(host="redis-5", port=6379),
]

QUORUM = len(nodes) // 2 + 1
LOCK_VALIDITY_MS = 10000
CLOCK_DRIFT_FACTOR = 0.01

def try_acquire(node, key, value, ttl_ms):
    try:
        return node.set(key, value, nx=True, px=ttl_ms)
    except redis.RedisError:
        return False

def release(node, key, value):
    RELEASE_SCRIPT = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    try:
        node.eval(RELEASE_SCRIPT, 1, key, value)
    except redis.RedisError:
        pass

def acquire_redlock(resource: str):
    lock_value = str(uuid.uuid4())
    start = time.time() * 1000

    acquired = sum(
        1 for node in nodes
        if try_acquire(node, resource, lock_value, LOCK_VALIDITY_MS)
    )

    elapsed = time.time() * 1000 - start
    drift = CLOCK_DRIFT_FACTOR * LOCK_VALIDITY_MS + 2
    validity = LOCK_VALIDITY_MS - elapsed - drift

    if acquired >= QUORUM and validity > 0:
        return lock_value, validity
    else:
        for node in nodes:
            release(node, resource, lock_value)
        return None, 0

def release_redlock(resource: str, lock_value: str):
    for node in nodes:
        release(node, resource, lock_value)

# Usage
lock_val, validity = acquire_redlock("lock:payment:order:5001")
if lock_val:
    try:
        print(f"Lock acquired, valid for {validity:.0f}ms")
        # critical section
    finally:
        release_redlock("lock:payment:order:5001", lock_val)
else:
    print("Failed to acquire lock")
```

## Lock Extension

Extend a lock that is close to expiry:

```python
def extend_lock(r, key, value, additional_ms: int):
    EXTEND_SCRIPT = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("pexpire", KEYS[1], ARGV[2])
    else
        return 0
    end
    """
    return r.eval(EXTEND_SCRIPT, 1, key, value, additional_ms)
```

## Summary

Python implements Redlock using either the `redis-py` built-in lock for single-node scenarios or the `redlock-py` library for multi-node fault tolerance. The key correctness properties are: using a random UUID as the lock value to prevent accidental release, setting a TTL to prevent deadlocks from crashed processes, and using a Lua script for atomic check-and-release. Always handle `LockNotOwnedError` when the lock expires before you can release it.
