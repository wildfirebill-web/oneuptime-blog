# How to Implement Read-Write Locks with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Read-Write Lock, Distributed Lock, Concurrency, Lua Script

Description: Learn how to implement read-write (shared-exclusive) locks in Redis to allow concurrent reads while ensuring exclusive access for write operations.

---

A read-write lock (also called a shared-exclusive lock) allows multiple concurrent readers but only one writer at a time. This is more efficient than a simple mutex when reads are frequent and writes are rare. Redis doesn't have built-in read-write locks, but they can be implemented using atomic operations.

## How Read-Write Locks Work

- **Read lock (shared)**: Multiple readers can hold it simultaneously. Blocked only by write locks.
- **Write lock (exclusive)**: Only one writer at a time. Blocked by both read and write locks.

## Implementation Using Redis Hashes

Store the lock state as a hash: one field for the write lock and a counter for readers.

```python
import redis
import uuid
import time

ACQUIRE_READ_SCRIPT = """
-- Acquire a read lock if no write lock is held
local write_lock = redis.call('HGET', KEYS[1], 'write')
if write_lock and write_lock ~= '' then
    return 0
end
redis.call('HINCRBY', KEYS[1], 'readers', 1)
redis.call('EXPIRE', KEYS[1], ARGV[1])
return 1
"""

RELEASE_READ_SCRIPT = """
-- Release a read lock
local count = redis.call('HINCRBY', KEYS[1], 'readers', -1)
if count <= 0 then
    redis.call('HDEL', KEYS[1], 'readers')
end
return 1
"""

ACQUIRE_WRITE_SCRIPT = """
-- Acquire a write lock only if no readers and no other writer
local readers = tonumber(redis.call('HGET', KEYS[1], 'readers')) or 0
local write_lock = redis.call('HGET', KEYS[1], 'write')
if readers > 0 or (write_lock and write_lock ~= '') then
    return 0
end
redis.call('HSET', KEYS[1], 'write', ARGV[1])
redis.call('EXPIRE', KEYS[1], ARGV[2])
return 1
"""

RELEASE_WRITE_SCRIPT = """
-- Release a write lock only if we own it
if redis.call('HGET', KEYS[1], 'write') == ARGV[1] then
    redis.call('HDEL', KEYS[1], 'write')
    return 1
end
return 0
"""

class ReadWriteLock:
    def __init__(self, r: redis.Redis, resource: str, ttl_seconds: int = 30):
        self.r = r
        self.key = f"rwlock:{resource}"
        self.ttl = ttl_seconds

    def acquire_read(self, timeout: float = 5.0) -> bool:
        deadline = time.time() + timeout
        while time.time() < deadline:
            result = self.r.eval(ACQUIRE_READ_SCRIPT, 1, self.key, self.ttl)
            if result == 1:
                return True
            time.sleep(0.05)
        return False

    def release_read(self):
        self.r.eval(RELEASE_READ_SCRIPT, 1, self.key)

    def acquire_write(self, timeout: float = 5.0) -> str | None:
        token = str(uuid.uuid4())
        deadline = time.time() + timeout
        while time.time() < deadline:
            result = self.r.eval(ACQUIRE_WRITE_SCRIPT, 1, self.key, token, self.ttl)
            if result == 1:
                return token
            time.sleep(0.05)
        return None

    def release_write(self, token: str) -> bool:
        result = self.r.eval(RELEASE_WRITE_SCRIPT, 1, self.key, token)
        return result == 1
```

## Using the Read-Write Lock

```python
r = redis.Redis(host='localhost', decode_responses=True)
rw_lock = ReadWriteLock(r, "config:app_settings")

# Multiple readers can proceed concurrently
def read_config():
    if rw_lock.acquire_read(timeout=2.0):
        try:
            config = r.hgetall("config:app_settings:data")
            return config
        finally:
            rw_lock.release_read()
    raise Exception("Could not acquire read lock")

# Only one writer at a time
def update_config(key: str, value: str):
    token = rw_lock.acquire_write(timeout=5.0)
    if token is None:
        raise Exception("Could not acquire write lock")
    try:
        r.hset("config:app_settings:data", key, value)
    finally:
        rw_lock.release_write(token)
```

## Checking Lock State

```python
def get_lock_state(r: redis.Redis, resource: str) -> dict:
    key = f"rwlock:{resource}"
    state = r.hgetall(key)
    return {
        "readers": int(state.get("readers", 0)),
        "write_locked": bool(state.get("write", "")),
        "ttl_seconds": r.ttl(key),
    }

# Example output
state = get_lock_state(r, "config:app_settings")
print(state)
```

Expected output when 3 readers are active:

```text
{'readers': 3, 'write_locked': False, 'ttl_seconds': 28}
```

## Summary

Redis read-write locks use a hash to track reader count and write lock ownership, with Lua scripts ensuring atomic transitions. Multiple readers can hold the lock simultaneously as long as no writer holds it. Acquire write locks only when no readers or other writers are present. Always release locks in `finally` blocks and set TTLs to prevent lock leaks from crashed clients.
