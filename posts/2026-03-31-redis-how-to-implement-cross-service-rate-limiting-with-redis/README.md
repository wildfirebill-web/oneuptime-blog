# How to Implement Cross-Service Rate Limiting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, Microservices, Distributed Systems, Node.js

Description: Implement shared Redis-backed rate limiting across multiple microservices so that rate limit counters are enforced globally regardless of which service handles the request.

---

## Why Cross-Service Rate Limiting

In a microservices architecture, rate limiting in each service independently leads to:

- Users hitting 10x the intended limit if they call 10 different services
- No coordination between services on per-user API budgets
- Inconsistent limits depending on which service handles the request

Redis provides a shared counter store that all services read from and write to atomically.

## Architecture

```text
Service A (e.g., /api/orders)  --|
Service B (e.g., /api/products)|-> Shared Redis counter -> Limit enforced globally
Service C (e.g., /api/search)  --|

All services increment the same counter keyed by user ID (and optionally plan).
```

## Shared Rate Limiter Module

Create a shared module that all services import:

```javascript
// shared/rateLimiter.js
const Redis = require('ioredis');

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
});

/**
 * Fixed window rate limiter
 * @param {string} identifier - User ID, API key, or IP address
 * @param {string} namespace - Rate limit scope (e.g., 'api', 'search')
 * @param {number} limit - Max requests allowed
 * @param {number} windowSeconds - Window size in seconds
 */
async function checkRateLimit(identifier, namespace, limit, windowSeconds) {
  const window = Math.floor(Date.now() / (windowSeconds * 1000));
  const key = `ratelimit:${namespace}:${identifier}:${window}`;

  const pipeline = redis.pipeline();
  pipeline.incr(key);
  pipeline.expire(key, windowSeconds);
  const [[, count]] = await pipeline.exec();

  const remaining = Math.max(0, limit - count);
  const allowed = count <= limit;

  return {
    allowed,
    count,
    limit,
    remaining,
    resetAt: (window + 1) * windowSeconds * 1000,
  };
}

module.exports = { checkRateLimit, redis };
```

## Service A: Orders Service

```javascript
// services/orders/middleware.js
const express = require('express');
const { checkRateLimit } = require('../../shared/rateLimiter');

const app = express();

async function rateLimitMiddleware(req, res, next) {
  const userId = req.headers['x-user-id'];
  if (!userId) {
    return res.status(401).json({ error: 'Authentication required' });
  }

  // Global API limit: 1000 requests per hour per user
  const result = await checkRateLimit(userId, 'api', 1000, 3600);

  res.set('X-RateLimit-Limit', result.limit.toString());
  res.set('X-RateLimit-Remaining', result.remaining.toString());
  res.set('X-RateLimit-Reset', new Date(result.resetAt).toISOString());

  if (!result.allowed) {
    return res.status(429).json({
      error: 'Rate limit exceeded',
      retryAfter: Math.ceil((result.resetAt - Date.now()) / 1000),
    });
  }

  next();
}

app.use(rateLimitMiddleware);
app.get('/orders', async (req, res) => {
  res.json({ orders: [] });
});
```

## Service B: Products Service (Same Shared Limit)

```javascript
// services/products/middleware.js
const express = require('express');
const { checkRateLimit } = require('../../shared/rateLimiter');

const app = express();

async function rateLimitMiddleware(req, res, next) {
  const userId = req.headers['x-user-id'];

  // Same 'api' namespace, same limit - shared counter with orders service
  const result = await checkRateLimit(userId, 'api', 1000, 3600);

  if (!result.allowed) {
    return res.status(429).json({ error: 'Global rate limit exceeded' });
  }

  next();
}

app.use(rateLimitMiddleware);
```

## Sliding Window Rate Limiter

For more accurate cross-service limiting without window boundary bursts:

```javascript
const slidingWindowLimiter = `
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window_ms = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local request_id = ARGV[4]

-- Remove old entries outside the window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window_ms)

-- Count current requests in window
local count = redis.call('ZCARD', key)

if count >= limit then
  local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
  if #oldest > 0 then
    return {0, count, tonumber(oldest[2]) + window_ms}
  end
  return {0, count, now + window_ms}
end

-- Add this request
redis.call('ZADD', key, now, request_id .. '-' .. now)
redis.call('EXPIRE', key, math.ceil(window_ms / 1000))
return {1, count + 1, 0}
`;

async function slidingRateLimit(identifier, namespace, limit, windowMs) {
  const key = `sliding:${namespace}:${identifier}`;
  const now = Date.now();
  const requestId = Math.random().toString(36).substring(7);

  const [allowed, count, resetAt] = await redis.eval(
    slidingWindowLimiter,
    1,
    key,
    limit,
    windowMs,
    now,
    requestId
  );

  return {
    allowed: allowed === 1,
    count,
    limit,
    remaining: Math.max(0, limit - count),
    resetAt: resetAt > 0 ? resetAt : null,
  };
}
```

## Multi-Tier Rate Limiting

Enforce multiple limits simultaneously (per-user AND per-plan):

```javascript
async function multiTierRateLimit(userId, planTier) {
  const tiers = {
    free: { limit: 100, window: 3600 },
    pro: { limit: 1000, window: 3600 },
    enterprise: { limit: 10000, window: 3600 },
  };

  const config = tiers[planTier] ?? tiers.free;

  // Check both user-specific and plan-level limits
  const [userLimit, planLimit] = await Promise.all([
    checkRateLimit(userId, 'user', config.limit, config.window),
    checkRateLimit(planTier, 'plan', config.limit * 10, config.window), // Plan-wide limit
  ]);

  // Enforce the most restrictive limit
  if (!userLimit.allowed || !planLimit.allowed) {
    return {
      allowed: false,
      reason: !userLimit.allowed ? 'user_limit_exceeded' : 'plan_limit_exceeded',
      userRemaining: userLimit.remaining,
      planRemaining: planLimit.remaining,
    };
  }

  return {
    allowed: true,
    userRemaining: userLimit.remaining,
    planRemaining: planLimit.remaining,
  };
}
```

## Health Monitoring

```javascript
async function getRateLimitMetrics(namespace) {
  let cursor = '0';
  let totalActiveKeys = 0;

  do {
    const [newCursor, keys] = await redis.scan(
      cursor, 'MATCH', `ratelimit:${namespace}:*`, 'COUNT', 100
    );
    cursor = newCursor;
    totalActiveKeys += keys.length;
  } while (cursor !== '0');

  return { namespace, activeKeys: totalActiveKeys };
}
```

## Summary

Cross-service rate limiting with Redis works by sharing a common namespace in Redis so that all services increment the same per-user counter. Use atomic Redis operations (pipeline INCR + EXPIRE or Lua scripts) to prevent race conditions. Implement multi-tier limits to enforce both per-user and per-plan boundaries simultaneously. The sliding window approach eliminates boundary bursts at the cost of slightly more complex Lua logic.
