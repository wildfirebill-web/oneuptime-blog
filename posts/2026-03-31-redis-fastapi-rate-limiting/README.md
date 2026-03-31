# How to Implement FastAPI Rate Limiting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, FastAPI, Rate Limiting, Python, Middleware

Description: Implement sliding window and fixed window rate limiting in FastAPI using Redis to protect your API from abuse and traffic spikes.

---

Rate limiting prevents abuse and protects backend resources. Redis is an ideal store for rate limit counters because reads and writes are fast, atomic operations are supported via Lua scripts, and keys expire automatically.

## Install Dependencies

```bash
pip install fastapi uvicorn redis
```

## Fixed Window with INCR and EXPIRE

The simplest approach increments a counter per client per time window:

```python
import redis
import time
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()
r = redis.Redis(host="localhost", port=6379, decode_responses=True)

RATE_LIMIT = 10   # requests
WINDOW_SEC = 60   # per minute

def check_rate_limit(client_id: str):
    key = f"rl:{client_id}:{int(time.time() // WINDOW_SEC)}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, WINDOW_SEC)
    if count > RATE_LIMIT:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
```

Use it in a route:

```python
@app.get("/api/data")
def get_data(request: Request):
    check_rate_limit(request.client.host)
    return {"data": "ok"}
```

## Sliding Window with a Lua Script

The fixed window allows a burst at the window boundary. A sliding window distributes requests evenly:

```lua
-- sliding_window.lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

redis.call("ZREMRANGEBYSCORE", key, 0, now - window * 1000)
local count = redis.call("ZCARD", key)

if count < limit then
    redis.call("ZADD", key, now, now)
    redis.call("PEXPIRE", key, window * 1000)
    return 1
else
    return 0
end
```

Load and call the script from Python:

```python
import time

with open("sliding_window.lua") as f:
    lua_script = f.read()

sliding_window = r.register_script(lua_script)

def sliding_rate_limit(client_id: str):
    now_ms = int(time.time() * 1000)
    key = f"sw:{client_id}"
    allowed = sliding_window(keys=[key], args=[now_ms, WINDOW_SEC, RATE_LIMIT])
    if not allowed:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
```

## FastAPI Middleware

Apply rate limiting globally with middleware:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

class RateLimitMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        client = request.client.host
        try:
            check_rate_limit(client)
        except HTTPException:
            return JSONResponse({"detail": "Rate limit exceeded"}, status_code=429)
        return await call_next(request)

app.add_middleware(RateLimitMiddleware)
```

## Return Rate Limit Headers

Help clients understand their quota:

```python
@app.get("/api/resource")
def resource(request: Request):
    key = f"rl:{request.client.host}:{int(time.time() // WINDOW_SEC)}"
    count = int(r.get(key) or 0)
    remaining = max(0, RATE_LIMIT - count)
    return JSONResponse(
        {"data": "ok"},
        headers={
            "X-RateLimit-Limit": str(RATE_LIMIT),
            "X-RateLimit-Remaining": str(remaining),
        },
    )
```

## Summary

Redis provides the atomic counters and automatic key expiry needed for reliable rate limiting in FastAPI. A fixed window counter is simple to implement, while a Lua-based sliding window prevents burst abuse at window boundaries. Applying the check in middleware ensures every route is protected without repetitive code.
