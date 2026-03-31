# How Redis Handles Client Reconnection After Failover

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Failover, Client

Description: Learn how Redis clients reconnect after Sentinel or Cluster failover - what client libraries do automatically, and how to handle reconnection in your application.

---

When Redis fails over to a new primary (via Sentinel or Cluster), existing client connections to the old primary break. How quickly your application recovers depends on how your client library handles reconnection and how well it integrates with the failover mechanism.

## What Happens During Failover

1. The primary goes down
2. Sentinel or Cluster detects the failure and promotes a replica
3. The old primary's connections return errors or time out
4. Clients must discover the new primary and reconnect

The critical window is between the old primary going down and clients connecting to the new one.

## Sentinel-Based Failover

Clients using Sentinel-aware libraries discover the new primary by querying Sentinels:

```python
from redis.sentinel import Sentinel

sentinel = Sentinel(
    [('sentinel1', 26379), ('sentinel2', 26379), ('sentinel3', 26379)],
    socket_timeout=0.5
)
master = sentinel.master_for('mymaster', socket_timeout=0.5, retry_on_timeout=True)
```

The client asks Sentinels for the current master address on each connection. If the primary changes, the next call transparently connects to the new one.

## Cluster-Based Failover

Redis Cluster clients use MOVED redirects and automatic slot mapping refresh:

```python
from redis.cluster import RedisCluster

rc = RedisCluster(
    host='node1',
    port=7000,
    retry_on_timeout=True,
    skip_full_coverage_check=True
)
```

On MOVED redirect, the client updates its slot map and retries the command against the correct node.

## Connection Retry Logic

Most operations during failover will fail with connection errors. Implement retry with backoff:

```python
import redis
from redis.exceptions import ConnectionError, TimeoutError
import time

def execute_with_retry(r, fn, max_retries=5):
    for i in range(max_retries):
        try:
            return fn(r)
        except (ConnectionError, TimeoutError) as e:
            if i == max_retries - 1:
                raise
            wait = min(0.1 * (2 ** i), 5.0)
            print(f"Retry {i+1} after {wait}s due to: {e}")
            time.sleep(wait)
```

## Testing Client Reconnection

Use `DEBUG SLEEP` to simulate a slow primary and test reconnection:

```bash
redis-cli DEBUG SLEEP 10
```

Or kill the primary process and verify your application reconnects within an acceptable time.

## Checking Connection Health

Monitor failed reconnection attempts in your application metrics. Track:
- Time to reconnect after failover
- Number of failed commands during failover window
- Connection pool exhaustion

## Tuning Timeout Parameters

Configure client timeouts to detect failures quickly without too many false positives:

```python
r = redis.Redis(
    host='localhost',
    port=6379,
    socket_connect_timeout=2,
    socket_timeout=2,
    retry_on_timeout=True,
    health_check_interval=30
)
```

## Summary

Client reconnection after Redis failover depends on whether the client uses Sentinel or Cluster APIs for primary discovery. Sentinel clients query Sentinels for the new primary address. Cluster clients follow MOVED redirects. Both require retry logic in the application to handle the brief window of command failures during failover transition.
