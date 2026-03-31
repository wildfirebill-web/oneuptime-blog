# How to Implement API Gateway Caching with Redis in Microservices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, API Gateway, Microservice, Caching, Node.js

Description: Build Redis-backed caching at the API gateway layer to reduce downstream microservice load and improve response times across your entire service mesh.

---

## API Gateway Caching Architecture

Caching at the API gateway provides a centralized cache that benefits all downstream microservices without requiring each service to implement its own caching layer.

```text
Client -> API Gateway (cache check) -> [cache hit] -> return cached response
                                    -> [cache miss] -> Upstream microservice
                                                   -> Cache response
                                                   -> Return to client
```

Benefits:
- Reduces total requests to downstream services
- Consistent caching policy across all endpoints
- Centralized TTL and cache invalidation management
- Works without modifying individual microservices

## Building a Caching Gateway Middleware

```javascript
const express = require('express');
const Redis = require('ioredis');
const crypto = require('crypto');
const httpProxy = require('http-proxy-middleware');

const redis = new Redis({ host: process.env.REDIS_HOST || 'localhost' });
const app = express();

const CACHE_TTL = parseInt(process.env.CACHE_TTL || '300');

function getCacheKey(req) {
  const hash = crypto.createHash('sha256')
    .update(`${req.method}:${req.path}:${JSON.stringify(req.query)}`)
    .digest('hex')
    .substring(0, 16);
  return `gateway:${req.path.replace(/\//g, ':')}:${hash}`;
}

function isCacheable(req, res) {
  // Only cache GET requests with successful responses
  return req.method === 'GET' && res.statusCode >= 200 && res.statusCode < 300;
}

async function cacheMiddleware(req, res, next) {
  if (req.method !== 'GET') {
    return next();
  }

  const cacheKey = getCacheKey(req);
  const cached = await redis.get(cacheKey);

  if (cached) {
    const { body, headers, statusCode } = JSON.parse(cached);
    res.set(headers);
    res.set('X-Cache', 'HIT');
    res.set('X-Cache-Key', cacheKey);
    return res.status(statusCode).send(body);
  }

  res.set('X-Cache', 'MISS');

  // Intercept the response to cache it
  const originalSend = res.send.bind(res);
  res.send = function (body) {
    if (isCacheable(req, res)) {
      const cacheData = {
        body,
        statusCode: res.statusCode,
        headers: {
          'Content-Type': res.get('Content-Type'),
        },
      };
      redis.setex(cacheKey, CACHE_TTL, JSON.stringify(cacheData)).catch(console.error);
    }
    return originalSend(body);
  };

  next();
}

app.use(cacheMiddleware);
```

## Route-Specific Cache Policies

```javascript
const cacheConfig = {
  '/api/products': { ttl: 600, varyBy: ['category', 'sort'] },
  '/api/users/:id': { ttl: 60, varyBy: [] },
  '/api/config': { ttl: 3600, varyBy: [] },
  '/api/search': { ttl: 120, varyBy: ['q', 'page', 'limit'] },
};

function getRouteCacheConfig(path) {
  for (const [pattern, config] of Object.entries(cacheConfig)) {
    if (matchPath(pattern, path)) {
      return config;
    }
  }
  return { ttl: CACHE_TTL, varyBy: [] };
}

function getCacheKeyWithVary(req, config) {
  const varyValues = config.varyBy.reduce((acc, param) => {
    acc[param] = req.query[param] ?? '';
    return acc;
  }, {});

  const hash = crypto.createHash('sha256')
    .update(`${req.path}:${JSON.stringify(varyValues)}`)
    .digest('hex')
    .substring(0, 16);

  return `gateway:${req.path}:${hash}`;
}
```

## Cache Invalidation on Mutations

```javascript
// Invalidation service called after mutating operations
const invalidationRules = {
  'POST /api/products': ['gateway:api:products:*'],
  'PUT /api/products/:id': ['gateway:api:products:*', 'gateway:api:products:{id}:*'],
  'DELETE /api/products/:id': ['gateway:api:products:*'],
};

async function invalidateCache(patterns) {
  for (const pattern of patterns) {
    let cursor = '0';
    do {
      const [newCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = newCursor;
      if (keys.length > 0) {
        await redis.del(...keys);
        console.log(`Invalidated ${keys.length} cache keys matching ${pattern}`);
      }
    } while (cursor !== '0');
  }
}

// Webhook or event listener for cache invalidation
app.post('/internal/cache/invalidate', async (req, res) => {
  const { event, entityType, entityId } = req.body;
  const patterns = buildInvalidationPatterns(entityType, entityId);
  await invalidateCache(patterns);
  res.json({ invalidated: true, patterns });
});
```

## Cache Stampede Protection (Probabilistic Early Expiration)

```javascript
async function getCachedWithStampedeProtection(cacheKey, fetchFn, ttl) {
  const raw = await redis.get(cacheKey);

  if (raw) {
    const { data, expiresAt } = JSON.parse(raw);
    const remaining = expiresAt - Date.now();

    // Probabilistic early refresh: if within 20% of TTL, sometimes refresh early
    const earlyRefreshWindow = ttl * 1000 * 0.2;
    if (remaining < earlyRefreshWindow && Math.random() < 0.1) {
      // 10% chance to refresh early, preventing all clients hitting DB at once
      fetchFn().then(freshData => {
        redis.setex(cacheKey, ttl, JSON.stringify({
          data: freshData,
          expiresAt: Date.now() + ttl * 1000,
        }));
      }).catch(console.error);
    }

    return data;
  }

  // Cache miss - use distributed lock to prevent stampede
  const lockKey = `lock:${cacheKey}`;
  const lock = await redis.set(lockKey, '1', 'EX', 5, 'NX');

  if (!lock) {
    // Another request is fetching - wait briefly and retry
    await new Promise(resolve => setTimeout(resolve, 100));
    return getCachedWithStampedeProtection(cacheKey, fetchFn, ttl);
  }

  try {
    const data = await fetchFn();
    await redis.setex(cacheKey, ttl, JSON.stringify({
      data,
      expiresAt: Date.now() + ttl * 1000,
    }));
    return data;
  } finally {
    await redis.del(lockKey);
  }
}
```

## Monitoring Cache Performance

```javascript
async function getCacheMetrics() {
  let cursor = '0';
  let totalKeys = 0;

  do {
    const [newCursor, keys] = await redis.scan(cursor, 'MATCH', 'gateway:*', 'COUNT', 100);
    cursor = newCursor;
    totalKeys += keys.length;
  } while (cursor !== '0');

  const info = await redis.info('stats');
  const hitsMatch = info.match(/keyspace_hits:(\d+)/);
  const missesMatch = info.match(/keyspace_misses:(\d+)/);

  const hits = parseInt(hitsMatch?.[1] ?? '0');
  const misses = parseInt(missesMatch?.[1] ?? '0');
  const total = hits + misses;
  const hitRate = total > 0 ? (hits / total * 100).toFixed(1) : 0;

  return { totalCacheKeys: totalKeys, cacheHitRate: `${hitRate}%`, hits, misses };
}
```

## Summary

API gateway caching with Redis provides a centralized layer that intercepts GET requests, returns cached responses for cache hits, and stores successful responses for cache misses. Implement route-specific TTL policies for different endpoint volatility levels, use distributed locking for cache stampede protection, and set up event-driven cache invalidation when upstream data changes. This pattern can reduce downstream microservice load by 60-90% for read-heavy APIs.
