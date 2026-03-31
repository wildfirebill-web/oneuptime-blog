# How to Handle Redis Replication Lag in Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Replication, Application, Lag, High Availability

Description: Learn strategies to handle Redis replication lag in your applications, including read-your-writes consistency, lag detection, and routing decisions.

---

Redis replication lag - the delay between a write on the primary and its visibility on replicas - is unavoidable in distributed systems. The key is building applications that tolerate or work around it gracefully.

## Detecting Replication Lag

Before handling lag, measure it. Use the `INFO replication` command on the primary:

```bash
redis-cli -h primary-host INFO replication
```

Look for `slave0: ip=...,port=...,state=online,offset=...,lag=N`. The `lag` field shows seconds of delay. You can also query the offset difference programmatically:

```python
import redis

primary = redis.Redis(host='primary', port=6379)
replica = redis.Redis(host='replica', port=6379)

def get_replication_lag_bytes():
    primary_info = primary.info('replication')
    replica_info = replica.info('replication')
    primary_offset = primary_info['master_repl_offset']
    replica_offset = replica_info['master_repl_offset']
    return primary_offset - replica_offset
```

## Strategy 1 - Read-Your-Writes via Sticky Routing

For operations where a user must immediately read what they wrote, route reads to the primary after a write:

```python
def save_user_profile(user_id, data):
    primary.hset(f'user:{user_id}', mapping=data)
    # Return a "read from primary" flag valid for 1 second
    primary.setex(f'read_primary:{user_id}', 1, '1')

def get_user_profile(user_id):
    if primary.exists(f'read_primary:{user_id}'):
        return primary.hgetall(f'user:{user_id}')
    return replica.hgetall(f'user:{user_id}')
```

## Strategy 2 - Lag-Aware Replica Selection

Only route reads to replicas that are sufficiently caught up:

```python
MAX_LAG_BYTES = 50000  # ~50KB behind is acceptable

def get_healthy_replica(replicas):
    primary_info = primary.info('replication')
    primary_offset = primary_info['master_repl_offset']

    for r in replicas:
        r_info = r.info('replication')
        lag = primary_offset - r_info['master_repl_offset']
        if lag <= MAX_LAG_BYTES:
            return r
    # Fall back to primary if no replica is healthy
    return primary
```

## Strategy 3 - Use WAIT for Critical Writes

For operations that cannot tolerate any replica lag, use the `WAIT` command:

```bash
SET critical_config "new_value"
WAIT 1 500
# waits for at least 1 replica to acknowledge, timeout 500ms
```

In Python:

```python
pipe = primary.pipeline()
pipe.set('critical_config', 'new_value')
pipe.execute()
# Ensure at least 1 replica has the write within 500ms
primary.wait(1, 500)
```

## Strategy 4 - Circuit Breaker for High Lag

If replication lag exceeds a threshold, stop routing to that replica entirely:

```python
import time

class ReplicaCircuitBreaker:
    def __init__(self, max_lag_seconds=5):
        self.max_lag = max_lag_seconds
        self.open_until = {}

    def is_healthy(self, replica_host):
        if replica_host in self.open_until:
            if time.time() < self.open_until[replica_host]:
                return False  # Still open
            del self.open_until[replica_host]

        lag = get_lag_for_replica(replica_host)
        if lag > self.max_lag:
            self.open_until[replica_host] = time.time() + 30
            return False
        return True
```

## Monitoring in Production

Track these metrics for alerting:

```bash
# Alert if lag exceeds 10 seconds
redis-cli INFO replication | grep lag
```

Consider monitoring replication lag with OneUptime to get alerts before it affects users. Set a threshold alert when `master_last_io_seconds_ago` exceeds your SLA tolerance.

## Summary

Handling replication lag in Redis applications requires detecting lag, routing critical reads to the primary, and optionally using `WAIT` for synchronous guarantees. Combining lag-aware routing with circuit breakers provides resilient read scaling without sacrificing consistency for critical operations.
