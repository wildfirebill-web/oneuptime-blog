# How to Implement Cost-Based Rate Limiting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, API, Performance, Lua

Description: Learn how to implement cost-based rate limiting with Redis, where each API operation consumes a variable number of tokens based on its computational weight.

---

Traditional rate limiting treats every request equally - one request deducts one token. Cost-based rate limiting is more nuanced: a lightweight read might cost 1 token while a heavy search query costs 10. Redis is an ideal backend for this pattern because of its atomic Lua scripting and sub-millisecond latency.

## Why Cost-Based Rate Limiting?

Standard counters protect against request volume but not resource exhaustion. An API client calling an expensive aggregation endpoint 100 times can be just as damaging as one calling a lightweight endpoint 10,000 times. Assigning costs per operation makes limits fair and resource-aware.

## Data Structure

Use a Redis key per client to store the remaining token budget and a TTL for the refill window.

```text
Key:   ratelimit:{client_id}
Value: remaining token budget (integer)
TTL:   window duration in seconds
```

## Atomic Deduction with Lua

Lua scripts execute atomically on a single Redis node, preventing race conditions between check and decrement.

```lua
-- cost_rate_limit.lua
local key = KEYS[1]
local cost = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local window = tonumber(ARGV[3])

local current = redis.call("GET", key)

if current == false then
  -- First request in this window
  redis.call("SET", key, limit - cost, "EX", window)
  return limit - cost
end

current = tonumber(current)

if current < cost then
  return -1  -- Insufficient budget
end

return redis.call("DECRBY", key, cost)
```

## Calling the Script from Python

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

script = r.register_script(open("cost_rate_limit.lua").read())

def check_rate_limit(client_id: str, operation_cost: int, limit: int = 1000, window: int = 60) -> bool:
    key = f"ratelimit:{client_id}"
    result = script(keys=[key], args=[operation_cost, limit, window])
    return int(result) >= 0

# Define operation costs
OPERATION_COSTS = {
    "list": 1,
    "search": 10,
    "export": 50,
    "aggregate": 25,
}

def handle_request(client_id: str, operation: str):
    cost = OPERATION_COSTS.get(operation, 1)
    if not check_rate_limit(client_id, cost):
        return {"error": "Rate limit exceeded", "retry_after": 60}
    return {"status": "ok"}
```

## Returning Budget Info in Headers

```python
def get_remaining_budget(client_id: str) -> int:
    key = f"ratelimit:{client_id}"
    val = r.get(key)
    return int(val) if val else 1000

# In your HTTP handler
remaining = get_remaining_budget(client_id)
response.headers["X-RateLimit-Remaining"] = str(remaining)
response.headers["X-RateLimit-Limit"] = "1000"
```

## Testing the Behavior

```bash
# Start Redis
docker run -d -p 6379:6379 redis:7

# Test via redis-cli
redis-cli SET ratelimit:user123 1000 EX 60
redis-cli DECRBY ratelimit:user123 50   # simulate export cost
redis-cli GET ratelimit:user123
```

## Monitoring with OneUptime

Track rate limit hit rates as custom metrics. Alert when more than 5% of requests are rejected within a 5-minute window, which can indicate abusive clients or misconfigured cost values.

## Summary

Cost-based rate limiting assigns variable token costs per operation, making limits proportional to resource usage rather than raw request count. Redis Lua scripts provide the atomicity needed to safely deduct variable amounts. Pair this with per-client headers to give API consumers clear feedback on their remaining budget.

