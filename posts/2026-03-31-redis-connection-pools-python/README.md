# How to Use Connection Pools in Python redis-py

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, Connection Pool, Performance, redis-py

Description: Configure and use Redis connection pools in Python with redis-py to reduce connection overhead, tune pool size, and share pools across your application.

---

Creating a new TCP connection for every Redis command is expensive. Connection pools maintain a set of reusable connections, reducing latency and preventing connection exhaustion under load. `redis-py` uses a connection pool by default, but understanding how to configure it is essential for production deployments.

## Default Pool Behavior

When you create a `Redis` instance, redis-py automatically creates a `ConnectionPool` behind the scenes:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
print(type(r.connection_pool))  # <class 'redis.connection.ConnectionPool'>
```

By default, the pool size is unlimited. Connections are created on demand and returned to the pool after each command.

## Creating an Explicit Connection Pool

```python
import redis

pool = redis.ConnectionPool(
    host="localhost",
    port=6379,
    db=0,
    max_connections=20,
    decode_responses=True,
)

r = redis.Redis(connection_pool=pool)
print(r.ping())  # True
```

Set `max_connections` to match your application's concurrency level. A web server with 8 workers and 4 threads each needs at most 32 connections.

## Sharing a Pool Across the Application

Define the pool once at module level and import it wherever you need Redis:

```python
# redis_client.py
import redis
import os

_pool = redis.ConnectionPool.from_url(
    os.getenv("REDIS_URL", "redis://localhost:6379/0"),
    max_connections=50,
    decode_responses=True,
)

def get_redis() -> redis.Redis:
    return redis.Redis(connection_pool=_pool)
```

```python
# app.py
from redis_client import get_redis

r = get_redis()
r.set("app:status", "running")
```

## Blocking Connection Pool

Use `BlockingConnectionPool` to raise an error (or wait) when the pool is exhausted rather than creating unlimited connections:

```python
import redis

pool = redis.BlockingConnectionPool(
    host="localhost",
    port=6379,
    max_connections=10,
    timeout=5,  # Wait up to 5s before raising ConnectionError
    decode_responses=True,
)

r = redis.Redis(connection_pool=pool)
```

This prevents unbounded connection growth on traffic spikes.

## Pool with TLS

```python
import ssl
import redis

pool = redis.ConnectionPool(
    host="my-redis.example.com",
    port=6380,
    connection_class=redis.SSLConnection,
    ssl_certfile="/path/to/client.crt",
    ssl_keyfile="/path/to/client.key",
    ssl_ca_certs="/path/to/ca.crt",
    max_connections=20,
    decode_responses=True,
)

r = redis.Redis(connection_pool=pool)
```

## Inspecting Pool State

```python
pool_stats = {
    "created_connections": pool._created_connections,
    "available_connections": len(pool._available_connections),
    "in_use": pool._created_connections - len(pool._available_connections),
}
print(pool_stats)
```

## Closing the Pool

In applications with a clean shutdown path, release all connections:

```python
import atexit
atexit.register(pool.disconnect)
```

## Summary

redis-py uses a connection pool by default. For production, create an explicit `ConnectionPool` with a bounded `max_connections`, share a single pool instance across your app, and consider `BlockingConnectionPool` to prevent connection exhaustion under load. Always close the pool on application shutdown to release TCP connections cleanly.
