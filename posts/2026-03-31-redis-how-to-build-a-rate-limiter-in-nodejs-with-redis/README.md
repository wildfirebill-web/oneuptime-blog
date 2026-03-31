# How to Build a Rate Limiter in Node.js with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Node.js, Rate Limiting, ioredis, Express, API Security

Description: Learn how to build a Redis-backed rate limiter in Node.js using fixed window, sliding window, and token bucket algorithms with Express middleware integration.

---

## Why Redis for Rate Limiting?

Redis is perfect for rate limiting because atomic operations prevent race conditions, TTL handles window expiry automatically, and it works across multiple Node.js instances for distributed rate limiting.

## Fixed Window Rate Limiter

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function isRateLimited(identifier, limit = 100, windowSeconds = 60) {
  const windowKey = Math.floor(Date.now() / 1000 / windowSeconds);
  const key = `rate:${identifier}:${windowKey}`;

  const pipeline = redis.pipeline();
  pipeline.incr(key);
  pipeline.expire(key, windowSeconds);
  const [[, count]] = await pipeline.exec();

  return {
    limited: count > limit,
    count,
    limit,
    remaining: Math.max(0, limit - count)
  };
}

// Test
const result = await isRateLimited('user:1001', 5, 60);
console.log(result);
```

## Sliding Window Log

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function slidingWindowCheck(identifier, limit = 100, windowMs = 60000) {
  const key = `rate:sliding:${identifier}`;
  const now = Date.now();
  const windowStart = now - windowMs;

  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(key, 0, windowStart);
  pipeline.zcard(key);
  pipeline.zadd(key, now, `${now}-${Math.random()}`);
  pipeline.expire(key, Math.ceil(windowMs / 1000) + 1);

  const results = await pipeline.exec();
  const count = results[1][1] + 1; // +1 for the just-added entry

  return {
    limited: count > limit,
    count,
    limit,
    remaining: Math.max(0, limit - count),
    resetAt: now + windowMs
  };
}
```

## Token Bucket

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function tokenBucketConsume(identifier, capacity = 10, refillRate = 1) {
  const key = `rate:tokens:${identifier}`;
  const now = Date.now() / 1000; // seconds

  for (let attempt = 0; attempt < 5; attempt++) {
    await redis.watch(key);

    const data = await redis.hgetall(key);
    let tokens = data.tokens ? parseFloat(data.tokens) : capacity;
    let lastRefill = data.last_refill ? parseFloat(data.last_refill) : now;

    // Refill tokens
    const elapsed = now - lastRefill;
    tokens = Math.min(capacity, tokens + elapsed * refillRate);

    if (tokens < 1) {
      await redis.unwatch();
      return {
        allowed: false,
        tokens: 0,
        retryAfter: Math.ceil((1 - tokens) / refillRate)
      };
    }

    tokens -= 1;

    const results = await redis
      .multi()
      .hset(key, 'tokens', tokens.toString(), 'last_refill', now.toString())
      .expire(key, Math.ceil(capacity / refillRate) + 60)
      .exec();

    if (results !== null) {
      return { allowed: true, tokens: Math.floor(tokens), remaining: Math.floor(tokens) };
    }
  }

  return { allowed: false, tokens: 0 };
}
```

## Express Middleware

```javascript
const express = require('express');
const Redis = require('ioredis');

const app = express();
const redis = new Redis();

function rateLimitMiddleware(options = {}) {
  const {
    limit = 60,
    windowSeconds = 60,
    keyFn = (req) => req.headers['x-api-key'] || req.ip,
    message = 'Too many requests'
  } = options;

  return async (req, res, next) => {
    const identifier = keyFn(req);
    const windowKey = Math.floor(Date.now() / 1000 / windowSeconds);
    const key = `rate:${identifier}:${windowKey}`;

    try {
      const pipeline = redis.pipeline();
      pipeline.incr(key);
      pipeline.expire(key, windowSeconds);
      const [[, count]] = await pipeline.exec();

      // Set rate limit headers
      res.setHeader('X-RateLimit-Limit', limit);
      res.setHeader('X-RateLimit-Remaining', Math.max(0, limit - count));
      res.setHeader('X-RateLimit-Reset', (Math.floor(Date.now() / 1000 / windowSeconds) + 1) * windowSeconds);

      if (count > limit) {
        return res.status(429).json({
          error: message,
          retryAfter: windowSeconds - (Math.floor(Date.now() / 1000) % windowSeconds)
        });
      }

      next();
    } catch (err) {
      console.error('Rate limiter error:', err);
      next(); // Fail open - allow request if Redis is down
    }
  };
}

// Global rate limit
app.use(rateLimitMiddleware({ limit: 100, windowSeconds: 60 }));

// Stricter limit for auth routes
app.use('/api/auth', rateLimitMiddleware({
  limit: 5,
  windowSeconds: 300,
  message: 'Too many authentication attempts'
}));

app.get('/api/data', (req, res) => {
  res.json({ data: 'Hello!' });
});

app.listen(3000);
```

## Per-User and Per-Tier Rate Limiting

```javascript
const Redis = require('ioredis');
const redis = new Redis();

const RATE_LIMITS = {
  free: { limit: 100, window: 3600 },
  pro: { limit: 1000, window: 3600 },
  enterprise: { limit: 10000, window: 3600 }
};

async function checkTieredLimit(userId, tier = 'free') {
  const { limit, window: windowSec } = RATE_LIMITS[tier] || RATE_LIMITS.free;
  const windowKey = Math.floor(Date.now() / 1000 / windowSec);
  const key = `rate:tier:${userId}:${windowKey}`;

  const pipeline = redis.pipeline();
  pipeline.incr(key);
  pipeline.expire(key, windowSec);
  const [[, count]] = await pipeline.exec();

  return {
    allowed: count <= limit,
    tier,
    limit,
    used: count,
    remaining: Math.max(0, limit - count)
  };
}

// Usage
const result = await checkTieredLimit('user:1001', 'pro');
if (!result.allowed) {
  console.log(`Rate limited: ${result.used}/${result.limit} requests used`);
}
```

## Summary

Building a rate limiter in Node.js with Redis combines atomic `INCR` + `EXPIRE` for fixed windows, sorted sets for accurate sliding windows, and WATCH-based optimistic locking for token buckets. Use Express middleware to enforce limits with proper `X-RateLimit-*` headers and `429` responses. Always fail open when Redis is unavailable to prevent your rate limiter from becoming a single point of failure.
