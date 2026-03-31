# How to Invalidate Cache When MongoDB Data Changes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Redis, Cache, Invalidation, Change Stream

Description: Implement automatic cache invalidation for Redis when MongoDB data changes using Change Streams, event-driven patterns, and TTL strategies.

---

## Introduction

Cache invalidation is famously one of the hardest problems in computer science. When MongoDB data changes - from your application, a background job, or a direct database edit - cached copies in Redis must be invalidated. This guide covers several invalidation strategies, from simple key deletion to event-driven invalidation via MongoDB Change Streams.

## Strategy 1: Explicit Key Deletion on Write

The simplest approach: delete the cache key whenever your application writes to MongoDB.

```javascript
async function updateProduct(productId, updates) {
  const db = mongo.db("myapp")

  // Update MongoDB
  await db.collection("products").updateOne(
    { _id: productId },
    { $set: { ...updates, updatedAt: new Date() } }
  )

  // Explicit cache invalidation
  await redis.del(`product:${productId}`)

  // Also invalidate list caches that might include this product
  const keys = await redis.keys("products:list:*")
  if (keys.length > 0) {
    await redis.del(keys)
  }
}
```

## Strategy 2: Tag-Based Invalidation

Group related cache keys under tags for bulk invalidation:

```javascript
async function setCacheWithTag(key, value, tag, ttl = 300) {
  await redis.setex(key, ttl, JSON.stringify(value))
  // Track this key under its tag
  await redis.sadd(`tag:${tag}`, key)
  await redis.expire(`tag:${tag}`, ttl + 60)
}

async function invalidateByTag(tag) {
  const keys = await redis.smembers(`tag:${tag}`)
  if (keys.length > 0) {
    await redis.del(keys)
  }
  await redis.del(`tag:${tag}`)
}

// Usage
await setCacheWithTag("product:123", product, "category:electronics")
await setCacheWithTag("products:list:electronics", list, "category:electronics")

// Invalidate everything tagged with this category
await invalidateByTag("category:electronics")
```

## Strategy 3: MongoDB Change Streams

Automatically invalidate cache when MongoDB documents change:

```javascript
async function watchAndInvalidate() {
  const db = mongo.db("myapp")
  const changeStream = db.collection("products").watch([], {
    fullDocument: "updateLookup"
  })

  changeStream.on("change", async (change) => {
    const docId = change.documentKey._id.toString()

    switch (change.operationType) {
      case "update":
      case "replace":
      case "delete":
        await redis.del(`product:${docId}`)
        console.log(`Invalidated cache for product:${docId}`)
        break
      case "insert":
        // Invalidate list caches that might be stale
        const listKeys = await redis.keys("products:list:*")
        if (listKeys.length > 0) await redis.del(listKeys)
        break
    }
  })

  changeStream.on("error", (err) => {
    console.error("Change stream error:", err)
    // Re-establish the change stream after a delay
    setTimeout(watchAndInvalidate, 5000)
  })
}
```

## Strategy 4: TTL-Based Expiry

For data that can tolerate brief staleness, rely purely on TTL expiry:

```javascript
// Short TTL for frequently changing data (60 seconds)
await redis.setex(`session:${userId}`, 60, JSON.stringify(sessionData))

// Longer TTL for stable data (1 hour)
await redis.setex(`user:profile:${userId}`, 3600, JSON.stringify(profile))
```

## Invalidation Patterns Comparison

```text
Pattern            | Consistency | Complexity | Use Case
-------------------|-------------|------------|---------------------------
Explicit deletion  | Strong      | Low        | Direct app writes
Tag-based          | Strong      | Medium     | Related entity groups
Change Streams     | Eventual    | High       | Multi-writer environments
TTL-only           | Eventual    | Minimal    | Tolerates brief staleness
```

## Summary

Choose your invalidation strategy based on how many systems write to MongoDB. For single-application writes, explicit key deletion is the simplest and most reliable approach. For environments where multiple services or admin tools can modify data, MongoDB Change Streams provide automatic cache invalidation without requiring every writer to know about the cache. Always combine strategies with a reasonable TTL as a safety net against missed invalidations.
