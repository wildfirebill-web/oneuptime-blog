# How to Implement Lock Renewal (Heartbeat) in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Distributed Lock, Lock Renewal, Heartbeat, Concurrency

Description: Learn how to extend Redis distributed lock TTLs using a heartbeat thread to prevent expiration during long-running operations while keeping deadlock protection.

---

Lock renewal (also called heartbeat or watchdog) extends a Redis lock's TTL periodically while the holder is still alive. This prevents the lock from expiring during legitimate long-running operations while maintaining deadlock protection if the client crashes.

## The Problem

A 30-second TTL works for most operations, but:
- Large batch jobs may take minutes
- Database migrations can be slow
- External API calls may have variable latency

You could set a very long TTL, but then a crashed client would hold the lock for too long, blocking all other processes.

## The Heartbeat Pattern

A background thread (watchdog) extends the lock TTL as long as the main thread is actively working:

```python
import redis
import threading
import time
import uuid
import logging

logger = logging.getLogger(__name__)

RENEW_SCRIPT = """
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('pexpire', KEYS[1], ARGV[2])
else
    return 0
end
"""

class HeartbeatLock:
    def __init__(self, r: redis.Redis, resource: str,
                 ttl_ms: int = 30000, renewal_interval_ms: int = 10000):
        self.r = r
        self.resource = resource
        self.ttl_ms = ttl_ms
        self.renewal_interval_s = renewal_interval_ms / 1000
        self.token = str(uuid.uuid4())
        self._acquired = False
        self._heartbeat_thread = None
        self._stop_event = threading.Event()

    def acquire(self, timeout_ms: int = 10000) -> bool:
        deadline = time.time() + (timeout_ms / 1000)
        while time.time() < deadline:
            result = self.r.set(self.resource, self.token, nx=True, px=self.ttl_ms)
            if result:
                self._acquired = True
                self._start_heartbeat()
                return True
            time.sleep(0.1)
        return False

    def _start_heartbeat(self):
        self._stop_event.clear()
        self._heartbeat_thread = threading.Thread(
            target=self._heartbeat_loop,
            name=f"heartbeat-{self.resource}",
            daemon=True
        )
        self._heartbeat_thread.start()

    def _heartbeat_loop(self):
        while not self._stop_event.wait(timeout=self.renewal_interval_s):
            try:
                result = self.r.eval(RENEW_SCRIPT, 1, self.resource, self.token, self.ttl_ms)
                if result == 1:
                    logger.debug(f"Lock renewed: {self.resource}")
                else:
                    logger.warning(f"Lock lost during renewal: {self.resource}")
                    self._stop_event.set()
            except redis.RedisError as e:
                logger.error(f"Heartbeat error for {self.resource}: {e}")

    def release(self):
        self._stop_event.set()
        if self._heartbeat_thread:
            self._heartbeat_thread.join(timeout=2)

        RELEASE_SCRIPT = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """
        result = self.r.eval(RELEASE_SCRIPT, 1, self.resource, self.token)
        self._acquired = False
        return result == 1

    def __enter__(self):
        if not self.acquire():
            raise RuntimeError(f"Could not acquire lock: {self.resource}")
        return self

    def __exit__(self, *args):
        self.release()
```

## Using the Heartbeat Lock

```python
r = redis.Redis(host='localhost', decode_responses=True)

# Context manager usage
with HeartbeatLock(r, "job:data-migration:v2", ttl_ms=30000) as lock:
    print("Running long migration...")
    time.sleep(60)  # The heartbeat keeps the lock alive
    print("Migration complete")

# Manual usage
lock = HeartbeatLock(r, "job:monthly-report", ttl_ms=30000)
if lock.acquire(timeout_ms=5000):
    try:
        generate_monthly_report()
    finally:
        lock.release()
else:
    print("Another instance is generating the report")
```

## Testing the Heartbeat

Verify the lock stays alive across multiple TTL periods:

```python
def test_lock_renewal():
    r = redis.Redis(host='localhost', decode_responses=True)
    lock = HeartbeatLock(r, "test:renewal", ttl_ms=5000, renewal_interval_ms=2000)

    assert lock.acquire()

    # Wait longer than the initial TTL
    time.sleep(8)

    # Lock should still exist due to heartbeat
    remaining = r.pttl("test:renewal")
    assert remaining > 0, f"Lock expired! Remaining TTL: {remaining}ms"

    lock.release()

    # Key should be gone after release
    assert r.get("test:renewal") is None
    print("Heartbeat test passed")
```

Expected output:

```text
Heartbeat test passed
```

## Summary

Lock renewal via heartbeat lets Redis distributed locks scale to long-running operations without sacrificing deadlock protection. The pattern works by running a daemon thread that periodically extends the lock TTL using a Lua script that only renews if the token still matches. Always stop the heartbeat thread in a `finally` block and join it before release to prevent token renewal after the main work is complete.
