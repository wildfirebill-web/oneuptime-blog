# How to Implement a Read-Write Lock with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lock, Concurrency

Description: Build a distributed read-write lock with Redis that allows concurrent reads but exclusive writes, improving throughput for read-heavy workloads.

---

A read-write lock (RWLock) allows many readers to hold the lock simultaneously, but a writer requires exclusive access. This dramatically improves throughput when reads far outnumber writes - common in caching, configuration, and shared state scenarios.

## Design

- Readers increment a counter key. If no writer is present, the read is allowed.
- Writers set an exclusive lock key. Writers wait until all readers finish.
- Both use TTLs to prevent deadlocks.

```python
import redis
import time
import uuid

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

READ_LOCK_TTL = 5000    # 5 seconds per reader
WRITE_LOCK_TTL = 10000  # 10 seconds for writer
WRITE_LOCK_KEY = "rwlock:{resource}:write"
READ_COUNT_KEY = "rwlock:{resource}:readers"
```

## Acquiring a Read Lock

```python
def acquire_read_lock(resource: str, timeout_ms: int = 2000) -> str | None:
    write_key = f"rwlock:{resource}:write"
    reader_key = f"rwlock:{resource}:readers"
    token = str(uuid.uuid4())
    deadline = time.time() + timeout_ms / 1000

    while time.time() < deadline:
        # No writer holding the lock
        if not r.exists(write_key):
            # Register as a reader
            pipe = r.pipeline()
            pipe.incr(reader_key)
            pipe.expire(reader_key, READ_LOCK_TTL // 1000 + 1)
            pipe.execute()

            # Double-check no writer snuck in
            if not r.exists(write_key):
                return token

            # Writer appeared - back off
            r.decr(reader_key)
        time.sleep(0.05)
    return None

def release_read_lock(resource: str, token: str):
    reader_key = f"rwlock:{resource}:readers"
    count = r.decr(reader_key)
    if count <= 0:
        r.delete(reader_key)
```

## Acquiring a Write Lock

```python
def acquire_write_lock(resource: str, timeout_ms: int = 5000) -> str | None:
    write_key = f"rwlock:{resource}:write"
    reader_key = f"rwlock:{resource}:readers"
    token = str(uuid.uuid4())
    deadline = time.time() + timeout_ms / 1000

    while time.time() < deadline:
        # Try to grab the exclusive write slot
        acquired = r.set(write_key, token, px=WRITE_LOCK_TTL, nx=True)
        if acquired:
            # Wait for all readers to finish
            reader_wait_deadline = time.time() + 2.0
            while time.time() < reader_wait_deadline:
                readers = int(r.get(reader_key) or 0)
                if readers <= 0:
                    return token  # Write lock held, no readers
                time.sleep(0.01)
            # Timed out waiting for readers - release write lock
            r.eval(
                "if redis.call('GET',KEYS[1])==ARGV[1] then return redis.call('DEL',KEYS[1]) else return 0 end",
                1, write_key, token
            )
        time.sleep(0.05)
    return None

def release_write_lock(resource: str, token: str) -> bool:
    write_key = f"rwlock:{resource}:write"
    result = r.eval(
        "if redis.call('GET',KEYS[1])==ARGV[1] then return redis.call('DEL',KEYS[1]) else return 0 end",
        1, write_key, token
    )
    return bool(result)
```

## Context Managers

```python
import contextlib

@contextlib.contextmanager
def read_lock(resource: str):
    token = acquire_read_lock(resource)
    if not token:
        raise TimeoutError(f"Could not acquire read lock on '{resource}'")
    try:
        yield
    finally:
        release_read_lock(resource, token)

@contextlib.contextmanager
def write_lock(resource: str):
    token = acquire_write_lock(resource)
    if not token:
        raise TimeoutError(f"Could not acquire write lock on '{resource}'")
    try:
        yield
    finally:
        release_write_lock(resource, token)
```

## Usage

```python
def read_config(config_key: str) -> dict:
    with read_lock(config_key):
        return r.hgetall(f"config:{config_key}")

def update_config(config_key: str, values: dict):
    with write_lock(config_key):
        r.hset(f"config:{config_key}", mapping=values)
```

## Summary

A Redis-based read-write lock improves throughput for read-heavy shared resources by allowing concurrent readers while enforcing exclusive writer access. TTLs on both the write lock and reader counter prevent deadlocks if a process crashes mid-operation.
