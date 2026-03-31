# How to Implement Cross-Service Rate Limiting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, Microservices, Distributed Systems, API

Description: Learn how to implement shared rate limiting across multiple microservices using Redis so that all services enforce a single consistent quota per user or API key.

---

## Why Cross-Service Rate Limiting

In a microservices architecture, each service may have its own local rate limiter. This creates a problem: if a user calls Service A 100 times and Service B 100 times, each service independently allows 100 requests, but the true total is 200. The user has exceeded the intended global quota.

Cross-service rate limiting uses Redis as a shared counter store, so all services contribute to and check the same quota.

```text
User Request
     |
     +--> Service A --> Redis (check/increment shared counter)
     +--> Service B --> Redis (check/increment shared counter)
     +--> Service C --> Redis (check/increment shared counter)
```

## Redis Data Model for Shared Counters

Use a fixed window counter with a shared key format:

```text
ratelimit:{user_id}:{window}
```

Where `window` is the current time bucket (e.g., current minute):

```bash
# Key for user 1001 in the current minute
ratelimit:1001:28531800
```

## Atomic Counter with Lua Script

Use a Lua script to atomically increment and check the counter:

```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local ttl = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, ttl)
end

if current > limit then
    return {0, current, limit}
else
    return {1, current, limit}
end
```

```bash
redis-cli EVAL "$(cat ratelimit.lua)" 1 "ratelimit:1001:$(date +%s | awk '{print int($1/60)}')" 100 60
```

Returns `{allowed, current_count, limit}`.

## Node.js Implementation

```javascript
const redis = require('redis');
const client = redis.createClient({ url: 'redis://localhost:6379' });

const RATE_LIMIT_SCRIPT = `
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local ttl = tonumber(ARGV[2])
  local current = redis.call('INCR', key)
  if current == 1 then
    redis.call('EXPIRE', key, ttl)
  end
  if current > limit then
    return {0, current, limit}
  else
    return {1, current, limit}
  end
`;

async function checkRateLimit(userId, limitPerMinute = 100) {
  const window = Math.floor(Date.now() / 60000);
  const key = `ratelimit:${userId}:${window}`;

  const result = await client.eval(RATE_LIMIT_SCRIPT, {
    keys: [key],
    arguments: [String(limitPerMinute), '60']
  });

  return {
    allowed: result[0] === 1,
    current: result[1],
    limit: result[2]
  };
}

// Middleware for Express
async function rateLimitMiddleware(req, res, next) {
  const userId = req.headers['x-user-id'];
  if (!userId) return res.status(401).json({ error: 'Missing user ID' });

  const result = await checkRateLimit(userId, 100);

  res.setHeader('X-RateLimit-Limit', result.limit);
  res.setHeader('X-RateLimit-Remaining', Math.max(0, result.limit - result.current));

  if (!result.allowed) {
    return res.status(429).json({ error: 'Rate limit exceeded' });
  }

  next();
}
```

## Python Implementation

```python
import redis
import time
import math

r = redis.Redis(host='localhost', port=6379)

RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local ttl = tonumber(ARGV[2])
local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, ttl)
end
if current > limit then
    return {0, current, limit}
else
    return {1, current, limit}
end
"""

rate_limit_sha = r.script_load(RATE_LIMIT_SCRIPT)

def check_rate_limit(user_id: str, limit: int = 100, window_seconds: int = 60) -> dict:
    window = math.floor(time.time() / window_seconds)
    key = f"ratelimit:{user_id}:{window}"

    result = r.evalsha(rate_limit_sha, 1, key, str(limit), str(window_seconds))

    return {
        "allowed": result[0] == 1,
        "current": result[1],
        "limit": result[2],
        "remaining": max(0, limit - result[1])
    }
```

## Sliding Window Rate Limiting

For smoother rate limiting without window boundary spikes, use a sliding window with a sorted set:

```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window_ms = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local min_score = now - window_ms

redis.call('ZREMRANGEBYSCORE', key, '-inf', min_score)
local count = redis.call('ZCARD', key)

if count >= limit then
    return {0, count}
end

redis.call('ZADD', key, now, now)
redis.call('PEXPIRE', key, window_ms)
return {1, count + 1}
```

```python
import time

SLIDING_WINDOW_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window_ms = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local min_score = now - window_ms

redis.call('ZREMRANGEBYSCORE', key, '-inf', min_score)
local count = redis.call('ZCARD', key)

if count >= limit then
    return {0, count}
end

redis.call('ZADD', key, now, now)
redis.call('PEXPIRE', key, window_ms)
return {1, count + 1}
"""

sha = r.script_load(SLIDING_WINDOW_SCRIPT)

def sliding_window_check(user_id: str, limit: int = 100, window_ms: int = 60000) -> dict:
    now = int(time.time() * 1000)
    key = f"swrl:{user_id}"
    result = r.evalsha(sha, 1, key, str(limit), str(window_ms), str(now))
    return {"allowed": result[0] == 1, "current": result[1]}
```

## Shared Rate Limits Across Service Tiers

You can define different limits per service tier while sharing the same Redis counter:

```python
TIER_LIMITS = {
    "free": 60,        # 60 requests per minute
    "pro": 500,        # 500 requests per minute
    "enterprise": 5000 # 5000 requests per minute
}

def check_tiered_limit(user_id: str, tier: str) -> dict:
    limit = TIER_LIMITS.get(tier, 60)
    return check_rate_limit(user_id, limit=limit)
```

## Returning Rate Limit Headers

Standard rate limit response headers help clients implement backoff:

```python
from flask import Flask, request, jsonify, g

app = Flask(__name__)

@app.before_request
def apply_rate_limit():
    user_id = request.headers.get('X-User-Id')
    if not user_id:
        return jsonify({"error": "Missing X-User-Id"}), 401

    result = check_rate_limit(user_id)
    g.rate_limit_result = result

    if not result["allowed"]:
        response = jsonify({"error": "Rate limit exceeded"})
        response.headers['Retry-After'] = '60'
        response.headers['X-RateLimit-Limit'] = result["limit"]
        response.headers['X-RateLimit-Remaining'] = 0
        return response, 429

@app.after_request
def add_rate_limit_headers(response):
    if hasattr(g, 'rate_limit_result'):
        r = g.rate_limit_result
        response.headers['X-RateLimit-Limit'] = r["limit"]
        response.headers['X-RateLimit-Remaining'] = r["remaining"]
    return response
```

## Summary

Cross-service rate limiting with Redis ensures that all microservices share a consistent quota per user or API key. By using atomic Lua scripts to increment and check Redis counters, you guarantee race-free enforcement even across multiple service instances. Fixed window counters offer simplicity while sliding window implementations avoid boundary spikes. Deploying shared rate limiting as a middleware or shared library ensures consistent enforcement across your entire service mesh.
