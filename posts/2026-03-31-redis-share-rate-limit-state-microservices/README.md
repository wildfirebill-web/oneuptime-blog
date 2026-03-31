# How to Share Rate Limit State Across Microservices with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, Microservice, Distributed System, Architecture

Description: Learn how to use Redis as a shared rate limit store so multiple microservices enforce a unified per-client request budget across your entire platform.

---

When each microservice maintains its own rate limit counters, a client can call Service A 100 times and Service B 100 times, effectively bypassing a 100-request-per-minute limit. A shared Redis store solves this by centralizing state while each service independently checks and increments the same keys.

## Architecture Overview

```text
Client --> API Gateway --> Service A  \
                      --> Service B   --> Redis (shared rate limit counters)
                      --> Service C  /
```

All services point to the same Redis instance (or cluster). Each service uses a shared key schema so counts are aggregated.

## Shared Key Schema

```text
ratelimit:global:{client_id}           # across all services
ratelimit:service:{service_name}:{client_id}  # per-service sub-limits
```

## Shared Lua Script

Deploy the same Lua script in each service or centralize it in a shared library.

```lua
-- shared_rate_limit.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call("INCR", key)
if current == 1 then
  redis.call("EXPIRE", key, window)
end

if current > limit then
  return -1
end

return limit - current
```

## Python Shared Client Library

```python
# ratelimit_client.py
import redis

_redis = None

def get_redis():
    global _redis
    if _redis is None:
        _redis = redis.Redis(host="redis.internal", port=6379, decode_responses=True)
    return _redis

_script = None

def _get_script():
    global _script
    if _script is None:
        lua = open("shared_rate_limit.lua").read()
        _script = get_redis().register_script(lua)
    return _script

GLOBAL_LIMIT = 200   # total requests per minute across all services
SERVICE_LIMIT = 100  # per-service limit per minute

def check_limits(client_id: str, service_name: str) -> bool:
    r = get_redis()
    script = _get_script()

    global_key = f"ratelimit:global:{client_id}"
    service_key = f"ratelimit:service:{service_name}:{client_id}"

    # Check global limit
    global_remaining = script(keys=[global_key], args=[GLOBAL_LIMIT, 60])
    if int(global_remaining) < 0:
        return False

    # Check service-level limit
    service_remaining = script(keys=[service_key], args=[SERVICE_LIMIT, 60])
    return int(service_remaining) >= 0
```

## Usage in Each Service

```python
# In service A, B, or C - same code, different service_name
from ratelimit_client import check_limits

def handle_request(client_id: str):
    if not check_limits(client_id, service_name="orders"):
        return {"error": "Rate limit exceeded", "status": 429}
    # proceed with business logic
    return {"status": "ok"}
```

## Inspecting Shared State

```bash
# View all keys for a client
redis-cli KEYS "ratelimit:*:client-abc"

# Check specific counters
redis-cli GET ratelimit:global:client-abc
redis-cli GET ratelimit:service:orders:client-abc
redis-cli TTL ratelimit:global:client-abc
```

## Handling Redis Failure Gracefully

```python
def check_limits_safe(client_id: str, service_name: str) -> bool:
    try:
        return check_limits(client_id, service_name)
    except redis.RedisError:
        # Fail open: allow request if Redis is unavailable
        return True
```

## Summary

Sharing rate limit state in Redis gives all microservices a consistent view of each client's usage. A common Lua script increments a shared key and checks thresholds atomically, preventing double-counting across services. Use a global key for platform-wide limits alongside per-service keys for finer-grained control.

