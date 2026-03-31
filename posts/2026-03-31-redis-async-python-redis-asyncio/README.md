# How to Use Async Redis in Python with redis.asyncio

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, Asyncio, Async, redis-py

Description: Use Redis asynchronously in Python with redis.asyncio, covering async connections, connection pools, pipelines, and integration with FastAPI and asyncio.

---

`redis-py` ships with a full async client under `redis.asyncio`. It mirrors the synchronous API but all commands are coroutines, making it ideal for async frameworks like FastAPI, aiohttp, and raw asyncio applications.

## Installation

The async client is included in the standard `redis` package:

```bash
pip install redis
```

## Basic Async Connection

```python
import asyncio
import redis.asyncio as aioredis

async def main():
    r = aioredis.Redis(host="localhost", port=6379, decode_responses=True)

    await r.set("greeting", "hello async")
    value = await r.get("greeting")
    print(value)  # hello async

    await r.aclose()

asyncio.run(main())
```

## Using an Async Connection Pool

```python
import redis.asyncio as aioredis

pool = aioredis.ConnectionPool.from_url(
    "redis://localhost:6379/0",
    max_connections=20,
    decode_responses=True,
)

async def get_redis() -> aioredis.Redis:
    return aioredis.Redis(connection_pool=pool)
```

Share this pool across your application. For FastAPI, initialize it at startup and close it on shutdown:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
import redis.asyncio as aioredis

pool: aioredis.ConnectionPool | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global pool
    pool = aioredis.ConnectionPool.from_url(
        "redis://localhost:6379/0",
        max_connections=50,
        decode_responses=True,
    )
    yield
    await pool.aclose()

app = FastAPI(lifespan=lifespan)

@app.get("/cache/{key}")
async def get_cache(key: str):
    r = aioredis.Redis(connection_pool=pool)
    value = await r.get(key)
    return {"key": key, "value": value}
```

## Async Pipelines

Pipelines batch multiple commands into a single round trip:

```python
async def set_many(r: aioredis.Redis, items: dict[str, str]):
    async with r.pipeline(transaction=False) as pipe:
        for key, value in items.items():
            pipe.set(key, value, ex=300)
        await pipe.execute()
```

## Async Pub/Sub

```python
async def subscribe_to_channel(channel: str):
    r = aioredis.Redis(host="localhost", port=6379, decode_responses=True)
    pubsub = r.pubsub()
    await pubsub.subscribe(channel)

    async for message in pubsub.listen():
        if message["type"] == "message":
            print(f"Received: {message['data']}")
```

## Concurrent Redis Calls

Use `asyncio.gather` to run multiple Redis commands in parallel:

```python
async def get_user_data(r: aioredis.Redis, user_id: str):
    profile, settings, permissions = await asyncio.gather(
        r.hgetall(f"user:{user_id}:profile"),
        r.hgetall(f"user:{user_id}:settings"),
        r.smembers(f"user:{user_id}:permissions"),
    )
    return {"profile": profile, "settings": settings, "permissions": permissions}
```

## Closing the Connection

Always close the connection or pool to avoid resource leaks:

```python
await r.aclose()        # Close a single connection
await pool.aclose()     # Close all connections in the pool
```

## Summary

`redis.asyncio` provides a drop-in async version of the redis-py API. Use `aioredis.ConnectionPool` for shared pooling, `async with r.pipeline()` for batched commands, and `asyncio.gather` to run concurrent Redis calls. Integrate with FastAPI using the lifespan context manager to manage pool lifecycle safely.
