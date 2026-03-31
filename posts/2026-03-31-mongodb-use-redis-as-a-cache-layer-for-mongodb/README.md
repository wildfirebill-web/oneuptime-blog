# How to Use Redis as a Cache Layer for MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Redis, Cache, Performance, Node.js

Description: Implement Redis as a caching layer in front of MongoDB to reduce database load and improve read latency with cache-aside and TTL-based strategies.

---

## Introduction

Redis paired with MongoDB is a classic combination for high-read applications. MongoDB handles durable, flexible document storage while Redis serves as an in-memory cache for frequently accessed data. This guide covers the cache-aside pattern, TTL management, and cache warming strategies.

## Architecture Overview

```text
Client -> Application -> Redis (cache hit) -> Response
                       -> Redis (cache miss) -> MongoDB -> Redis (store) -> Response
```

## Basic Cache-Aside in Node.js

```javascript
const { MongoClient } = require("mongodb")
const Redis = require("ioredis")

const mongo = new MongoClient(process.env.MONGODB_URI)
const redis = new Redis(process.env.REDIS_URL)

const CACHE_TTL = 300 // 5 minutes

async function getUserById(userId) {
  const cacheKey = `user:${userId}`

  // Check Redis first
  const cached = await redis.get(cacheKey)
  if (cached) {
    return JSON.parse(cached)
  }

  // Cache miss - query MongoDB
  const db = mongo.db("myapp")
  const user = await db.collection("users").findOne({ _id: userId })

  if (user) {
    // Store in Redis with TTL
    await redis.setex(cacheKey, CACHE_TTL, JSON.stringify(user))
  }

  return user
}

async function updateUser(userId, updates) {
  const db = mongo.db("myapp")
  await db.collection("users").updateOne(
    { _id: userId },
    { $set: { ...updates, updatedAt: new Date() } }
  )

  // Invalidate the cache
  await redis.del(`user:${userId}`)
}
```

## Python Implementation

```python
import json
import redis
from pymongo import MongoClient
import os

mongo_client = MongoClient(os.environ["MONGODB_URI"])
redis_client = redis.Redis.from_url(os.environ["REDIS_URL"])

CACHE_TTL = 300

def get_product(product_id: str):
    cache_key = f"product:{product_id}"

    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    db = mongo_client.myapp
    product = db.products.find_one({"_id": product_id}, {"_id": 0})

    if product:
        redis_client.setex(cache_key, CACHE_TTL, json.dumps(product, default=str))

    return product
```

## Caching Query Results

Cache entire query results, not just individual documents:

```javascript
async function getActiveProducts(category) {
  const cacheKey = `products:active:${category}`

  const cached = await redis.get(cacheKey)
  if (cached) return JSON.parse(cached)

  const products = await mongo
    .db("myapp")
    .collection("products")
    .find({ category, active: true })
    .sort({ price: 1 })
    .toArray()

  // Cache list results with a shorter TTL
  await redis.setex(cacheKey, 60, JSON.stringify(products))
  return products
}
```

## Cache Warming on Startup

```javascript
async function warmCache() {
  console.log("Warming product cache...")
  const db = mongo.db("myapp")
  const topProducts = await db
    .collection("products")
    .find({})
    .sort({ views: -1 })
    .limit(100)
    .toArray()

  const pipeline = redis.pipeline()
  for (const product of topProducts) {
    pipeline.setex(`product:${product._id}`, CACHE_TTL, JSON.stringify(product))
  }
  await pipeline.exec()
  console.log(`Warmed ${topProducts.length} products`)
}
```

## Summary

Redis as a MongoDB cache layer reduces latency for read-heavy workloads by serving frequent requests from memory. The cache-aside pattern is the most flexible approach: check Redis first, fall back to MongoDB on a miss, and always invalidate the cache key on writes. Use Redis pipelines for bulk operations, set appropriate TTLs based on data freshness requirements, and consider cache warming on application startup for the most frequently accessed records.
