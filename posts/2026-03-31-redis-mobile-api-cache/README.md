# How to Use Redis as Backend Cache for Mobile APIs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Mobile, API Caching

Description: Learn how to use Redis to cache mobile API responses, implement per-user caching, and handle cache invalidation for high-traffic mobile backends.

---

Mobile applications often make the same API requests repeatedly - user profiles, product catalogs, configuration. Redis caching at the API layer reduces database load, improves response times, and handles traffic spikes during app launches or promotional events.

## Cache API Responses in Express

```javascript
const express = require("express");
const Redis = require("ioredis");

const app = express();
const redis = new Redis({ host: "localhost", port: 6379 });

// Generic cache middleware
function cacheMiddleware(ttl = 300) {
  return async (req, res, next) => {
    if (req.method !== "GET") return next();

    // Include user ID in key for authenticated endpoints
    const userId = req.headers["x-user-id"] || "public";
    const cacheKey = `mobile:api:${userId}:${req.path}`;

    const cached = await redis.get(cacheKey);
    if (cached) {
      res.setHeader("X-Cache", "HIT");
      res.setHeader("X-Cache-TTL", await redis.ttl(cacheKey));
      return res.json(JSON.parse(cached));
    }

    // Capture the response
    const originalJson = res.json.bind(res);
    res.json = async (data) => {
      if (res.statusCode === 200) {
        await redis.set(cacheKey, JSON.stringify(data), "EX", ttl);
      }
      res.setHeader("X-Cache", "MISS");
      return originalJson(data);
    };

    next();
  };
}

// Apply to routes
app.get("/api/v1/user/profile", cacheMiddleware(120), async (req, res) => {
  const user = await db.getUserProfile(req.userId);
  res.json(user);
});

app.get("/api/v1/catalog", cacheMiddleware(600), async (req, res) => {
  const products = await db.getProducts();
  res.json(products);
});
```

## Version-Aware Cache Keys for App Releases

Different app versions may need different response shapes:

```javascript
function versionedCacheKey(req) {
  const appVersion = req.headers["x-app-version"] || "1.0.0";
  const major = appVersion.split(".")[0];
  return `mobile:v${major}:${req.path}`;
}
```

## Cache with ETag Support for Conditional Requests

Mobile clients can send `If-None-Match` to avoid downloading unchanged data:

```javascript
const crypto = require("crypto");

app.get("/api/v1/catalog", async (req, res) => {
  const cacheKey = "mobile:catalog";
  const etagKey = "mobile:catalog:etag";

  const [cached, etag] = await redis.mget(cacheKey, etagKey);

  if (cached && etag) {
    const clientEtag = req.headers["if-none-match"];
    if (clientEtag === etag) {
      return res.status(304).end(); // Not Modified
    }

    res.setHeader("ETag", etag);
    res.setHeader("X-Cache", "HIT");
    return res.json(JSON.parse(cached));
  }

  const products = await db.getProducts();
  const data = JSON.stringify(products);
  const newEtag = crypto.createHash("md5").update(data).digest("hex");

  const pipeline = redis.pipeline();
  pipeline.set(cacheKey, data, "EX", 600);
  pipeline.set(etagKey, newEtag, "EX", 600);
  await pipeline.exec();

  res.setHeader("ETag", newEtag);
  res.json(products);
});
```

## Invalidate User Cache After Profile Update

```javascript
async function invalidateUserCache(userId) {
  const pattern = `mobile:api:${userId}:*`;
  const keys = await redis.keys(pattern);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
}

app.put("/api/v1/user/profile", async (req, res) => {
  await db.updateProfile(req.userId, req.body);
  await invalidateUserCache(req.userId);
  res.json({ success: true });
});
```

## Summary

Redis mobile API caching intercepts GET requests, serves cached responses for cache hits, and stores new responses for cache misses. Use per-user cache keys for authenticated endpoints, include the app version in keys to support multiple client versions simultaneously, and implement ETag-based conditional requests to minimize data transfer for unchanged resources.
