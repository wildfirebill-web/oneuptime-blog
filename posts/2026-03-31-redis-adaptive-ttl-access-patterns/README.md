# How to Implement Adaptive TTL Based on Access Patterns in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, TTL, Adaptive, Access Pattern

Description: Learn how to implement adaptive TTL in Redis that extends a key's expiry on each access, keeping hot data alive longer while letting cold data expire naturally.

---

Static TTLs are a compromise: set them too short and hot data churns unnecessarily; set them too long and stale data lingers. Adaptive TTL adjusts expiry based on actual access: frequently accessed keys stay alive longer, rarely accessed keys expire on schedule.

## Strategy: Extend TTL on Read

Every time a key is read, reset its TTL to the maximum. Keys that are never read naturally expire.

```python
import redis
import json
from typing import Optional

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

MIN_TTL = 60       # seconds
MAX_TTL = 3600     # seconds
BASE_TTL = 300     # default TTL for new entries

def adaptive_get(key: str) -> Optional[dict]:
    raw = r.get(key)
    if raw is None:
        return None

    # Extend TTL on access - key is hot, keep it alive
    r.expire(key, MAX_TTL)
    return json.loads(raw)

def adaptive_set(key: str, value: dict, ttl: int = BASE_TTL):
    r.set(key, json.dumps(value), ex=ttl)
```

## Strategy 2: Tiered Adaptive TTL

Track access count and escalate TTL tier based on access frequency.

```python
ACCESS_COUNT_PREFIX = "access_count:"

TTL_TIERS = [
    (0, 60),      # 0 accesses: 1 min
    (5, 300),     # 5+ accesses: 5 min
    (20, 1800),   # 20+ accesses: 30 min
    (100, 7200),  # 100+ accesses: 2 hours
]

def get_adaptive_ttl(access_count: int) -> int:
    ttl = TTL_TIERS[0][1]
    for threshold, tier_ttl in TTL_TIERS:
        if access_count >= threshold:
            ttl = tier_ttl
    return ttl

def tiered_get(key: str) -> Optional[dict]:
    raw = r.get(key)
    if raw is None:
        return None

    # Increment access count
    count_key = f"{ACCESS_COUNT_PREFIX}{key}"
    access_count = r.incr(count_key)
    r.expire(count_key, 86400)  # reset access count daily

    # Set adaptive TTL based on access frequency
    new_ttl = get_adaptive_ttl(int(access_count))
    r.expire(key, new_ttl)

    return json.loads(raw)

def tiered_set(key: str, value: dict):
    count_key = f"{ACCESS_COUNT_PREFIX}{key}"
    existing_count = int(r.get(count_key) or 0)
    ttl = get_adaptive_ttl(existing_count)
    r.set(key, json.dumps(value), ex=ttl)
```

## Strategy 3: Sliding Window TTL via Lua

Atomically get and extend TTL in a single round trip.

```lua
-- adaptive_get.lua
local key = KEYS[1]
local max_ttl = tonumber(ARGV[1])

local value = redis.call("GET", key)
if value then
  redis.call("EXPIRE", key, max_ttl)
end
return value
```

```python
adaptive_script = r.register_script(open("adaptive_get.lua").read())

def lua_adaptive_get(key: str) -> Optional[dict]:
    raw = adaptive_script(keys=[key], args=[MAX_TTL])
    return json.loads(raw) if raw else None
```

## Inspecting Adaptive TTL State

```bash
# Current TTL
redis-cli TTL cache:user:u1

# Access count
redis-cli GET access_count:cache:user:u1

# Peek at value without resetting TTL
redis-cli GET cache:user:u1
```

## Cold Data Detection

```python
def find_cold_keys(pattern: str = "cache:*", max_ttl: int = 120) -> list:
    cold_keys = []
    for key in r.scan_iter(pattern):
        ttl = r.ttl(key)
        if 0 < ttl <= max_ttl:
            cold_keys.append({"key": key, "ttl": ttl})
    return sorted(cold_keys, key=lambda x: x["ttl"])
```

## Summary

Adaptive TTL keeps hot data alive by resetting expiry on each access, while cold data expires naturally without intervention. The tiered approach escalates TTL based on access count thresholds, giving very hot keys hours of survival while infrequently accessed keys expire in minutes. Use a Lua script for atomic get-and-extend to avoid race conditions under high concurrency.

