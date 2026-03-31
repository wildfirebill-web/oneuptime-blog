# How to Build a Serverless Rate Limiter with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Serverless, Rate Limiting

Description: Build a serverless rate limiter using Redis and atomic Lua scripts to enforce per-user request quotas in AWS Lambda or any serverless function platform.

---

Rate limiting serverless functions requires a shared counter store because each function instance runs independently. Redis is the natural choice - it supports atomic increment operations and TTL-based key expiry, making it perfect for sliding window and fixed window rate limiting.

## Fixed Window Rate Limiting

The simplest approach: count requests per time window with an auto-expiring key.

```javascript
const { createClient } = require('redis');

const client = createClient({ url: process.env.REDIS_URL });
await client.connect();

async function checkRateLimit(identifier, limit, windowSeconds) {
  const key = `rl:${identifier}:${Math.floor(Date.now() / 1000 / windowSeconds)}`;

  const count = await client.incr(key);
  if (count === 1) {
    await client.expire(key, windowSeconds);
  }

  return {
    allowed: count <= limit,
    remaining: Math.max(0, limit - count),
    resetAt: (Math.floor(Date.now() / 1000 / windowSeconds) + 1) * windowSeconds
  };
}

exports.handler = async (event) => {
  const userId = event.headers['x-user-id'] || event.requestContext?.identity?.sourceIp;
  const result = await checkRateLimit(userId, 100, 60);

  if (!result.allowed) {
    return {
      statusCode: 429,
      headers: {
        'X-RateLimit-Limit': '100',
        'X-RateLimit-Remaining': '0',
        'Retry-After': String(result.resetAt - Math.floor(Date.now() / 1000))
      },
      body: JSON.stringify({ error: 'Rate limit exceeded' })
    };
  }

  // Handle actual request
  return { statusCode: 200, body: JSON.stringify({ ok: true }) };
};
```

## Sliding Window with Lua Script

For more accurate rate limiting, use a sorted set to track request timestamps:

```javascript
const slidingWindowScript = `
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window * 1000)
local count = redis.call('ZCARD', key)

if count < limit then
  redis.call('ZADD', key, now, now)
  redis.call('PEXPIRE', key, window * 1000)
  return {1, limit - count - 1}
else
  return {0, 0}
end
`;

async function slidingWindowLimit(identifier, limit, windowSeconds) {
  const key = `slw:${identifier}`;
  const now = Date.now();

  const result = await client.eval(
    slidingWindowScript,
    { keys: [key], arguments: [String(now), String(windowSeconds), String(limit)] }
  );

  return {
    allowed: result[0] === 1,
    remaining: result[1]
  };
}
```

## Token Bucket Pattern

Token buckets allow short bursts while enforcing average rates:

```javascript
const tokenBucketScript = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refillRate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local lastRefill = tonumber(data[2]) or now

local elapsed = (now - lastRefill) / 1000
local newTokens = math.min(capacity, tokens + elapsed * refillRate)

if newTokens >= 1 then
  redis.call('HSET', key, 'tokens', newTokens - 1, 'last_refill', now)
  redis.call('EXPIRE', key, 3600)
  return 1
else
  return 0
end
`;
```

## Summary

Redis enables accurate serverless rate limiting through atomic operations - INCR with EXPIRE for simple fixed windows, sorted sets for sliding windows, and hash-based token buckets for burst-tolerant limits. Using Lua scripts ensures atomicity across multi-command sequences, preventing race conditions when multiple function instances update the same counter simultaneously.
