# How to Handle Lock Expiration in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Distributed Lock, Expiration, Timeout, Redis

Description: Learn how to handle Dapr lock expiration gracefully, implement lock renewal for long operations, and detect when your critical section outlived its lock TTL.

---

Every Dapr distributed lock has an expiry. When a lock expires before the owner releases it, another instance may acquire the same lock. This creates a race condition if the original owner is still running. Handling expiration correctly is critical for safe distributed coordination.

## Understanding What Happens on Expiry

When a lock expires:
1. Redis automatically deletes the lock key (TTL expires)
2. Another instance can immediately acquire the same lock
3. The original owner does not receive any notification
4. If the original owner tries to unlock, it will fail (wrong owner or lock gone)

## Detecting That Your Lock Expired

Check the unlock response to detect expiry:

```python
from dapr.clients import DaprClient
from dapr.proto.runtime.v1.dapr_pb2 import UnlockResponse

with DaprClient() as client:
    # Acquire
    lock = client.try_lock("lockstore", "my-resource", POD_NAME, 30)

    if lock.success:
        try:
            # Long operation...
            process()
        finally:
            result = client.unlock("lockstore", "my-resource", POD_NAME)
            if result.status != 0:  # 0 = success
                print(f"WARN: Unlock failed - lock may have expired during processing!")
                # Take corrective action - check for duplicate processing
```

## Implementing Lock Renewal (Heartbeat)

For operations that may exceed the lock TTL, renew the lock periodically:

```python
import threading
import time

class LockRenewer:
    def __init__(self, client, store, resource, owner, expiry):
        self.client = client
        self.store = store
        self.resource = resource
        self.owner = owner
        self.expiry = expiry
        self.active = True
        self._thread = None

    def start(self):
        self._thread = threading.Thread(target=self._renew_loop, daemon=True)
        self._thread.start()

    def stop(self):
        self.active = False

    def _renew_loop(self):
        interval = self.expiry * 0.6  # Renew at 60% of TTL
        while self.active:
            time.sleep(interval)
            if not self.active:
                break
            resp = self.client.try_lock(
                self.store, self.resource, self.owner, self.expiry
            )
            if not resp.success:
                print("WARNING: Failed to renew lock - another instance may have taken over")
                self.active = False

# Usage
with DaprClient() as client:
    lock = client.try_lock("lockstore", "long-job", POD_NAME, 60)
    if lock.success:
        renewer = LockRenewer(client, "lockstore", "long-job", POD_NAME, 60)
        renewer.start()
        try:
            run_long_job()
        finally:
            renewer.stop()
            client.unlock("lockstore", "long-job", POD_NAME)
```

## Setting Appropriate Expiry Values

Calculate expiry based on your operation's expected duration with a safety margin:

```python
def calculate_expiry(expected_duration_seconds: int, safety_factor: float = 2.0) -> int:
    return int(expected_duration_seconds * safety_factor)

# For an operation expected to take 10 seconds:
expiry = calculate_expiry(10)  # returns 20 seconds
```

## Guarding Against Post-Expiry Side Effects

Check that the lock is still valid before committing irreversible operations:

```python
def safe_commit(client, resource, owner):
    # Re-verify we still hold the lock before committing
    result = client.unlock("lockstore", resource, owner)
    if result.status != 0:
        # Lock expired - roll back any in-progress changes
        rollback()
        raise Exception("Lock expired before commit - operation rolled back")
    # Proceed with commit
    commit()
```

## Alerting on Frequent Expirations

Track unlock failures to surface expiry issues:

```python
from prometheus_client import Counter

lock_expiry_counter = Counter("lock_expiry_total", "Locks that expired before explicit release", ["resource"])

def monitored_unlock(client, store, resource, owner):
    result = client.unlock(store, resource, owner)
    if result.status != 0:
        lock_expiry_counter.labels(resource=resource).inc()
```

## Summary

Lock expiration in Dapr is automatic and silent - the owner receives no callback when its lock expires. Handle this by implementing a lock renewer for long operations, checking unlock responses for failure, and adding monitoring for expiry events. Setting expiry at 2x the expected operation duration provides a reasonable safety margin without causing excessive lock hold times.
