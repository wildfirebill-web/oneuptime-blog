# How to Use WAITAOF in Redis to Wait for AOF Persistence

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, AOF, Persistence, Durability, Data Safety

Description: Learn how to use WAITAOF in Redis to block until write operations are fsynced to disk, ensuring durability guarantees for critical data.

---

## What Is WAITAOF?

`WAITAOF` is a Redis command introduced in Redis 7.2 that blocks the client until the Append-Only File (AOF) has been fsynced to disk on a specified number of local and replica instances. It provides a way to confirm that data written to Redis has been durably persisted before continuing.

This command is useful in scenarios where you need strong durability guarantees - for example, after writing financial records, configuration changes, or other critical data that must survive a server crash.

## Basic Syntax

```text
WAITAOF numlocal numreplicas timeout
```

Parameters:
- `numlocal` - number of local AOF fsync confirmations to wait for (0 or 1)
- `numreplicas` - number of replica AOF fsync confirmations to wait for
- `timeout` - maximum time to wait in milliseconds (0 means wait indefinitely)

Returns an array with two integers: the number of local fsyncs confirmed and the number of replica fsyncs confirmed.

## Waiting for Local AOF Fsync

```bash
# Write some data
SET critical_config "value123"

# Wait for local AOF fsync (up to 5000ms)
WAITAOF 1 0 5000
# Returns: 1, 0  (1 local fsync confirmed, 0 replicas)
```

## Waiting for Replica AOF Fsync

```bash
SET payment_record "txn-9876"

# Wait for 2 replicas to confirm AOF fsync within 3 seconds
WAITAOF 0 2 3000
# Returns: 0, 2  (0 local, 2 replica fsyncs confirmed)
```

## Waiting for Both Local and Replicas

```bash
SET important_event "order-12345"

# Wait for local + 1 replica, up to 10 seconds
WAITAOF 1 1 10000
# Returns: 1, 1
```

## Timeout Behavior

If the timeout expires before the required number of fsyncs are confirmed, `WAITAOF` returns however many were confirmed up to that point. The caller should check the return values.

```bash
# Example where only 1 of 3 replicas confirmed in time
WAITAOF 1 3 1000
# Might return: 1, 1  (only 1 replica confirmed before timeout)
```

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

r.set('order_id', '42')

result = r.waitaof(1, 1, 5000)
local_fsyncs = result[0]
replica_fsyncs = result[1]

if local_fsyncs >= 1 and replica_fsyncs >= 1:
    print("Data durably persisted locally and on replica")
else:
    print(f"Only got: local={local_fsyncs}, replicas={replica_fsyncs}")
```

## WAITAOF vs WAIT

Both `WAIT` and `WAITAOF` block for replication confirmation, but they differ:

| Feature | WAIT | WAITAOF |
|---|---|---|
| Confirms replication | Yes | Yes |
| Confirms AOF fsync | No | Yes |
| Durability guarantee | Replication lag | Full disk fsync |
| Available since | 3.0 | 7.2 |

Use `WAIT` when you need to confirm data has been replicated to replicas. Use `WAITAOF` when you need to confirm data has been fsynced to disk.

## AOF Configuration Prerequisites

`WAITAOF` is only meaningful when AOF is enabled. Ensure your Redis configuration includes:

```text
appendonly yes
appendfsync everysec
```

Or for maximum durability:

```text
appendonly yes
appendfsync always
```

## Practical Use Case: Critical Write Pattern

```python
import redis

r = redis.Redis(host='localhost', port=6379)

def write_critical_data(key, value):
    """Write data and confirm AOF persistence before returning."""
    r.set(key, value)

    # Wait up to 5 seconds for local AOF fsync
    result = r.waitaof(1, 0, 5000)
    local_confirmed = result[0]

    if local_confirmed < 1:
        raise RuntimeError(f"AOF fsync not confirmed for key: {key}")

    return True

write_critical_data('user:1:balance', '1000')
```

## Summary

`WAITAOF` provides a powerful mechanism for ensuring that Redis write operations have been durably persisted to disk via AOF fsync before the application proceeds. It is most valuable in use cases requiring strong durability guarantees such as financial systems, configuration management, and audit logging. Always combine `WAITAOF` with a reasonable timeout and check the returned confirmation counts to handle partial persistence scenarios gracefully.
