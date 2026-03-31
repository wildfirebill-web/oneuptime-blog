# How to Design a Distributed Lock Using Redis in a System Design Interview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, System Design, Distributed Lock, Interview, Concurrency

Description: Design a distributed locking system with Redis covering SET NX PX, lock expiry, Redlock algorithm, fencing tokens, and failure modes in interviews.

---

## Why Distributed Locks

In a distributed system, multiple service instances may try to perform the same exclusive operation simultaneously - cron job execution, payment processing, inventory deduction. A distributed lock ensures only one instance proceeds at a time.

A correct distributed lock must be:
1. **Mutually exclusive**: only one holder at a time
2. **Deadlock-free**: lock expires automatically if the holder crashes
3. **Fault-tolerant**: works correctly under network partitions and node failures

## Requirements Clarification

**Functional:**
- Acquire a named lock with a timeout
- Release the lock (only the holder can release)
- Lock auto-expires if not released

**Non-functional:**
- Correctness over performance
- 10K lock acquisitions/sec
- Lock TTL between 1 second and 5 minutes
- Tolerate single Redis node failure (optional: Redlock)

## Single-Node Redis Lock

### Acquiring the Lock

Use `SET NX PX` to atomically set a key only if it does not exist, with a TTL:

```bash
SET lock:payment:order-1001 "unique-token-xyz" NX PX 30000
```

- `NX`: set only if key does Not eXist
- `PX 30000`: expire in 30 seconds (30000ms)
- Value `"unique-token-xyz"`: random token that identifies the lock holder

Returns `OK` if acquired, `nil` if the lock is already held.

### Releasing the Lock

The lock value must be a unique token per acquire attempt. Use a Lua script to atomically check-and-delete - this prevents accidentally releasing a lock held by a different client:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

```python
RELEASE_SCRIPT = """
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
"""

def release_lock(r, lock_name: str, token: str) -> bool:
    result = r.eval(RELEASE_SCRIPT, 1, f"lock:{lock_name}", token)
    return result == 1
```

Without the Lua script, a race condition exists: between `GET` and `DEL`, the lock could expire and be acquired by another client.

## Full Python Implementation

```python
import redis
import uuid
import time

r = redis.Redis(host='localhost', port=6379)

ACQUIRE_SCRIPT = "return redis.call('SET', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2])"
RELEASE_SCRIPT = """
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
"""

class RedisLock:
    def __init__(self, r, lock_name: str, ttl_ms: int = 30000):
        self.r = r
        self.key = f"lock:{lock_name}"
        self.token = str(uuid.uuid4())
        self.ttl_ms = ttl_ms
        self._acquired = False

    def acquire(self, retry_count: int = 3, retry_delay_ms: int = 200) -> bool:
        for attempt in range(retry_count):
            result = self.r.set(self.key, self.token, nx=True, px=self.ttl_ms)
            if result:
                self._acquired = True
                return True
            if attempt < retry_count - 1:
                time.sleep(retry_delay_ms / 1000)
        return False

    def release(self) -> bool:
        if not self._acquired:
            return False
        result = self.r.eval(RELEASE_SCRIPT, 1, self.key, self.token)
        self._acquired = False
        return result == 1

    def __enter__(self):
        if not self.acquire():
            raise RuntimeError(f"Could not acquire lock: {self.key}")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()


# Usage
with RedisLock(r, "payment:order-1001", ttl_ms=30000):
    process_payment(order_id=1001)
```

## Lock Extension (Watchdog Pattern)

If the operation might take longer than the TTL, extend the lock periodically using a watchdog thread:

```python
import threading

class WatchdogLock(RedisLock):
    def __init__(self, r, lock_name, ttl_ms=30000):
        super().__init__(r, lock_name, ttl_ms)
        self._watchdog = None

    def _extend_lock(self):
        while self._acquired:
            time.sleep(self.ttl_ms / 2000)  # extend at half the TTL interval
            if self._acquired:
                self.r.pexpire(self.key, self.ttl_ms)

    def acquire(self, **kwargs):
        result = super().acquire(**kwargs)
        if result:
            self._watchdog = threading.Thread(target=self._extend_lock, daemon=True)
            self._watchdog.start()
        return result

    def release(self):
        self._acquired = False
        return super().release()
```

## Failure Modes to Discuss in Interviews

### 1. Lock Holder Crashes Before Releasing

The TTL ensures automatic expiry. Next acquire attempt after TTL succeeds.

### 2. Clock Drift

If Redis server clock drifts, the effective TTL may be shorter than expected. Use conservative TTLs and account for drift in the safety margin.

### 3. Network Partition

If the application cannot reach Redis to release the lock, it expires automatically after TTL. The effective safety period is `TTL - (network_latency + clock_drift)`.

### 4. Redis Node Failure

With a single Redis node, if the node fails after granting a lock, the lock is lost (no expiry happens until node recovers). This can cause two processes to both believe they hold the lock.

## Redlock: Multi-Node Distributed Lock

For stronger guarantees, acquire the lock on N independent Redis nodes (typically 5) and consider it acquired if a majority (3 of 5) succeed within the validity time:

```python
import time
import uuid

class Redlock:
    def __init__(self, nodes: list[redis.Redis], quorum: int = None):
        self.nodes = nodes
        self.quorum = quorum or (len(nodes) // 2 + 1)

    def acquire(self, lock_name: str, ttl_ms: int = 30000) -> dict | None:
        token = str(uuid.uuid4())
        key = f"lock:{lock_name}"
        start_ms = int(time.time() * 1000)
        acquired = 0

        for node in self.nodes:
            try:
                if node.set(key, token, nx=True, px=ttl_ms):
                    acquired += 1
            except Exception:
                pass  # node unavailable

        elapsed_ms = int(time.time() * 1000) - start_ms
        validity_ms = ttl_ms - elapsed_ms - 10  # 10ms drift buffer

        if acquired >= self.quorum and validity_ms > 0:
            return {"key": key, "token": token, "validity_ms": validity_ms}

        # Failed to acquire majority - release on all nodes
        self._release_all(key, token)
        return None

    def _release_all(self, key: str, token: str):
        for node in self.nodes:
            try:
                node.eval(RELEASE_SCRIPT, 1, key, token)
            except Exception:
                pass
```

**Redlock trade-offs:**
- Safer against single-node failures
- More complex to operate (5 independent Redis nodes)
- Does not eliminate all failure modes under network partitions (Martin Kleppmann critique)

For most use cases, a single Redis node with proper TTLs and the check-and-delete release pattern is sufficient.

## Fencing Tokens for Extra Safety

Even with distributed locks, a paused process can wake up after its lock expired and perform operations. Add a monotonic fencing token to protect downstream systems:

```python
def acquire_with_token(r, lock_name: str, ttl_ms: int) -> dict | None:
    token = str(uuid.uuid4())
    fence_token = r.incr(f"fence:{lock_name}")  # monotonically increasing
    key = f"lock:{lock_name}"

    if r.set(key, token, nx=True, px=ttl_ms):
        return {"token": token, "fence": fence_token}
    return None
```

The downstream service rejects requests with a fence token lower than the last seen fence token.

## Summary

A Redis distributed lock uses `SET NX PX` for atomic acquire with TTL, and a Lua check-and-delete script for safe release. The unique token per acquire prevents accidentally releasing another client's lock. For multi-node safety, Redlock acquires on a majority of independent Redis nodes. In a system design interview, clearly articulate the failure modes - clock drift, network partitions, process pauses - and trade-offs between single-node simplicity and multi-node safety. Fencing tokens provide an additional layer of correctness for operations that cannot tolerate stale lock holders.
