# How to Handle Redis Connection Exhaustion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Connection Management, Performance

Description: Learn how to diagnose and fix Redis connection exhaustion, including pool sizing, leak detection, and graceful handling when the pool is full.

---

Redis connection exhaustion happens when all connections in your pool are in use and new requests cannot obtain one. This causes timeouts, errors, and request failures - often at the worst possible moment, during traffic spikes.

## Detect Connection Exhaustion

Check the current number of connected clients and compare against `maxclients`:

```bash
redis-cli INFO clients
```

```text
connected_clients:512
cluster_connections:0
maxclients:512
client_recent_max_input_buffer:20504
blocked_clients:0
```

When `connected_clients` approaches `maxclients`, you are near exhaustion. You can also check pool-level errors in your application logs for messages like "Connection pool exhausted" or "Timeout waiting for connection from pool."

## Increase maxclients

The Redis default is 10,000. Increase it in `redis.conf`:

```text
maxclients 20000
```

Or at runtime:

```bash
redis-cli CONFIG SET maxclients 20000
```

Also increase the OS file descriptor limit:

```bash
ulimit -n 65535
```

## Fix Pool Sizing in Python

```python
import redis

# Too small - causes exhaustion under load
pool = redis.ConnectionPool(host="localhost", port=6379, max_connections=10)

# Better - size to concurrency needs
# Rule of thumb: max_connections = (threads or coroutines) * 1.5
pool = redis.ConnectionPool(
    host="localhost",
    port=6379,
    max_connections=100,
    socket_connect_timeout=2,
    socket_timeout=2,
)

client = redis.Redis(connection_pool=pool)
```

## Detect Connection Leaks

Connections that are acquired and never released exhaust the pool over time. Use context managers to ensure release:

```python
# Bad: connection may not be returned if an exception occurs
conn = pool.get_connection("_")
conn.send_command("GET", "key")
# If exception here, connection leaks

# Good: context manager guarantees release
with client.pipeline() as pipe:
    pipe.get("key1")
    pipe.get("key2")
    results = pipe.execute()
```

## Handle Pool Exhaustion Gracefully

```python
from redis.exceptions import ConnectionError
import logging

logger = logging.getLogger(__name__)

def safe_redis_get(client, key, fallback=None):
    try:
        return client.get(key)
    except ConnectionError as e:
        logger.error(f"Redis pool exhausted for key {key}: {e}")
        # Fall back to database or return cached/default value
        return fallback
```

## Monitor with oneuptime

Track the `connected_clients` metric continuously:

```bash
# Export to a monitoring endpoint
watch -n 5 'redis-cli INFO clients | grep connected_clients'
```

Set an alert when `connected_clients` exceeds 80% of `maxclients` to give yourself time to react before exhaustion occurs.

## Summary

Redis connection exhaustion stems from pool sizes too small for the workload or connection leaks. Diagnose by watching `connected_clients` vs `maxclients`, size your pool to your application's concurrency level, use context managers to prevent leaks, and implement graceful degradation so the application can serve requests even when Redis is temporarily unavailable.
