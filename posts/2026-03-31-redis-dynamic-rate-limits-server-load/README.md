# How to Implement Dynamic Rate Limits Based on Server Load with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, Load Management, Performance, Python

Description: Learn how to dynamically adjust Redis rate limits based on real-time server load, reducing allowed throughput when your system is under pressure.

---

Static rate limits protect against abuse but they cannot adapt to varying infrastructure capacity. When your servers are under heavy load, even staying within a normal request rate can cause cascading failures. Dynamic rate limiting reads live load metrics and tightens the throttle automatically.

## The Core Idea

Store a current rate limit multiplier in Redis. A background process monitors CPU and memory and writes a new multiplier every few seconds. The rate-limiting path reads this multiplier before checking the counter.

```text
Key: system:load_multiplier  (float, e.g. 0.5 means half the normal limit)
Key: ratelimit:{client_id}   (request counter, standard token bucket or counter)
```

## Writing the Load Multiplier

```python
import psutil
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def compute_multiplier() -> float:
    cpu = psutil.cpu_percent(interval=1)
    mem = psutil.virtual_memory().percent
    # Scale down limit as load increases
    load_score = max(cpu, mem)
    if load_score < 50:
        return 1.0
    elif load_score < 70:
        return 0.75
    elif load_score < 85:
        return 0.5
    else:
        return 0.25

def update_multiplier():
    while True:
        multiplier = compute_multiplier()
        r.set("system:load_multiplier", multiplier, ex=30)
        time.sleep(5)
```

## Applying the Multiplier in Rate Limit Checks

```lua
-- dynamic_rate_limit.lua
local counter_key = KEYS[1]
local multiplier_key = KEYS[2]
local base_limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local multiplier = tonumber(redis.call("GET", multiplier_key) or "1.0")
local effective_limit = math.floor(base_limit * multiplier)

local current = tonumber(redis.call("GET", counter_key) or "0")

if current >= effective_limit then
  return -1
end

redis.call("INCR", counter_key)
redis.call("EXPIRE", counter_key, window)
return effective_limit - current - 1
```

## Wiring It Together in Python

```python
script = r.register_script(open("dynamic_rate_limit.lua").read())

BASE_LIMIT = 100  # requests per minute

def is_allowed(client_id: str) -> bool:
    counter_key = f"ratelimit:{client_id}"
    remaining = script(
        keys=[counter_key, "system:load_multiplier"],
        args=[BASE_LIMIT, 60]
    )
    return int(remaining) >= 0

def middleware(client_id: str, handler):
    if not is_allowed(client_id):
        return {"error": "Server under load, please retry later", "code": 503}
    return handler()
```

## Testing Under Simulated Load

```bash
# Simulate high CPU via redis-cli
redis-cli SET system:load_multiplier 0.25

# Now the effective limit is 25 instead of 100
redis-cli SET ratelimit:testclient 0
for i in $(seq 1 30); do redis-cli INCR ratelimit:testclient; done
redis-cli GET ratelimit:testclient
```

## Returning Load Context in Responses

```python
def get_load_info() -> dict:
    multiplier = float(r.get("system:load_multiplier") or 1.0)
    return {
        "x-ratelimit-multiplier": str(multiplier),
        "x-ratelimit-effective-limit": str(int(BASE_LIMIT * multiplier)),
    }
```

## Summary

Dynamic rate limiting uses a Redis-stored load multiplier updated by a background process to shrink allowed throughput when server resources are strained. A Lua script atomically reads the multiplier and increments the counter in one round trip. This approach prevents overload failures while allowing full capacity when the system is healthy.

