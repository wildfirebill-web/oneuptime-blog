# How to Use Redis for Next.js API Route Caching

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Next.js, Cache, API Route, Node.js

Description: Cache Next.js API route responses in Redis to reduce database load and improve response times for expensive or frequently-accessed endpoints.

---

Next.js API routes run on Node.js and have full access to npm packages. Caching their responses in Redis dramatically reduces database load for read-heavy endpoints that return data unlikely to change on every request.

## Install Dependencies

```bash
npm install redis
```

## Create a Redis Client Singleton

`lib/redis.js`:

```javascript
import { createClient } from "redis";

let client;

export async function getRedis() {
  if (!client) {
    client = createClient({ url: process.env.REDIS_URL || "redis://localhost:6379" });
    client.on("error", (err) => console.error("Redis error:", err));
    await client.connect();
  }
  return client;
}
```

## Cache an API Route Response

`pages/api/products.js`:

```javascript
import { getRedis } from "../../lib/redis";
import { fetchProducts } from "../../lib/db";

const CACHE_TTL = 300; // 5 minutes

export default async function handler(req, res) {
  const redis = await getRedis();
  const cacheKey = `api:products:${req.query.category || "all"}`;

  // Try cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    res.setHeader("X-Cache", "HIT");
    return res.json(JSON.parse(cached));
  }

  // Cache miss - fetch from DB
  const products = await fetchProducts(req.query.category);

  await redis.setEx(cacheKey, CACHE_TTL, JSON.stringify(products));

  res.setHeader("X-Cache", "MISS");
  res.json(products);
}
```

## Cache with Stale-While-Revalidate Pattern

Return stale data immediately while refreshing in the background:

```javascript
export default async function handler(req, res) {
  const redis = await getRedis();
  const key = "api:stats";

  const cached = await redis.get(key);
  if (cached) {
    const { data, timestamp } = JSON.parse(cached);
    const age = (Date.now() - timestamp) / 1000;

    // Return stale data and refresh in background if older than 60 seconds
    if (age > 60) {
      fetchAndCache(redis, key).catch(console.error); // Non-blocking
    }
    return res.json(data);
  }

  const data = await fetchAndCache(redis, key);
  res.json(data);
}

async function fetchAndCache(redis, key) {
  const data = await expensiveDbQuery();
  await redis.setEx(key, 300, JSON.stringify({ data, timestamp: Date.now() }));
  return data;
}
```

## Invalidate Cache on Mutation

```javascript
// pages/api/products/[id].js (PUT handler)
export default async function handler(req, res) {
  if (req.method === "PUT") {
    await updateProduct(req.query.id, req.body);
    const redis = await getRedis();
    // Invalidate all product cache variants
    const keys = await redis.keys("api:products:*");
    if (keys.length > 0) await redis.del(keys);
    return res.json({ updated: true });
  }
}
```

## Summary

Redis caching in Next.js API routes reduces database queries to near zero for popular endpoints by returning cached JSON directly. A singleton Redis client avoids connection overhead on every request, and the stale-while-revalidate pattern keeps responses fast even during cache refresh. Cache invalidation on write mutations ensures clients never see indefinitely stale data.
