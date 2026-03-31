# How to Use Redis with Starlette in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Starlette, Python, Caching, Async

Description: Learn how to connect Redis to a Starlette application for async caching, middleware rate limiting, and session storage with practical code examples.

---

Starlette is a lightweight ASGI framework that serves as the foundation for FastAPI and other Python web tools. Its async-first design makes it an excellent match for Redis. This guide demonstrates how to integrate Redis into a Starlette application.

## Installing Dependencies

```bash
pip install starlette uvicorn redis
```

## Setting Up Redis with Lifespan

Starlette recommends using the lifespan context manager to manage resource lifecycles.

```python
from contextlib import asynccontextmanager
from starlette.applications import Starlette
from starlette.routing import Route
from starlette.responses import JSONResponse
import redis.asyncio as aioredis

redis_client = None

@asynccontextmanager
async def lifespan(app):
    global redis_client
    redis_client = aioredis.from_url(
        "redis://localhost:6379",
        decode_responses=True,
    )
    await redis_client.ping()
    print("Redis connected")
    yield
    await redis_client.aclose()
    print("Redis disconnected")
```

## Caching in Route Handlers

```python
import json

async def get_product(request):
    product_id = request.path_params["product_id"]
    cache_key = f"product:{product_id}"
    cached = await redis_client.get(cache_key)
    if cached:
        return JSONResponse(json.loads(cached), headers={"X-Cache": "HIT"})

    product = {"id": product_id, "name": "Widget"}
    await redis_client.setex(cache_key, 120, json.dumps(product))
    return JSONResponse(product, headers={"X-Cache": "MISS"})
```

## Rate Limiting Middleware

Starlette's middleware API makes it easy to implement request-level rate limiting.

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import PlainTextResponse

class RateLimitMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        ip = request.client.host
        key = f"ratelimit:{ip}"
        count = await redis_client.incr(key)
        if count == 1:
            await redis_client.expire(key, 60)
        if count > 100:
            return PlainTextResponse("Rate limit exceeded", status_code=429)
        return await call_next(request)
```

## Session Storage

Store session tokens in Redis for stateless server deployments.

```python
import uuid

async def login(request):
    body = await request.json()
    session_id = str(uuid.uuid4())
    await redis_client.setex(
        f"session:{session_id}",
        86400,
        json.dumps({"user": body.get("username")}),
    )
    response = JSONResponse({"message": "Logged in"})
    response.set_cookie("session_id", session_id, httponly=True)
    return response

async def profile(request):
    session_id = request.cookies.get("session_id")
    if not session_id:
        return JSONResponse({"error": "Unauthorized"}, status_code=401)
    raw = await redis_client.get(f"session:{session_id}")
    if not raw:
        return JSONResponse({"error": "Session expired"}, status_code=401)
    return JSONResponse(json.loads(raw))
```

## Wiring It All Together

```python
from starlette.routing import Route
from starlette.middleware import Middleware

routes = [
    Route("/products/{product_id}", get_product),
    Route("/login", login, methods=["POST"]),
    Route("/profile", profile),
]

middleware = [
    Middleware(RateLimitMiddleware),
]

app = Starlette(
    lifespan=lifespan,
    routes=routes,
    middleware=middleware,
)
```

Run the app:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

## Summary

Starlette integrates with Redis through the async `redis.asyncio` client. Use the lifespan context manager for connection management, `BaseHTTPMiddleware` for rate limiting, and `setex` for TTL-controlled caching and session storage.
