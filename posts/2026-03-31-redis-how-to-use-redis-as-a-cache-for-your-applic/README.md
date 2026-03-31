# How to Use Redis as a Cache for Your Application

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Caching, Cache-Aside, Performance, Beginner, Node.js, Python

Description: A beginner-friendly guide to using Redis as an application cache, covering the cache-aside pattern, TTL management, and cache invalidation basics.

---

## What Is a Cache?

A cache stores the results of expensive operations (database queries, API calls, computations) so that future requests can be served from memory instead of recomputing them. Redis, as an in-memory key-value store, is one of the most popular caching solutions.

## Why Redis for Caching?

- Sub-millisecond read latency
- Simple key-value API
- Built-in TTL (time-to-live) for automatic expiration
- Supports complex data types (Hashes, Lists, Sets)

## Installing Redis Clients

```bash
# Node.js
npm install ioredis

# Python
pip install redis

# Ruby
gem install redis
```

## The Cache-Aside Pattern

This is the most common caching pattern:

1. Check Redis first
2. If cache hit, return cached data
3. If cache miss, fetch from database, store in Redis, return data

```text
Request
  |
  v
Check Redis --> Cache Hit --> Return data
  |
  | Cache Miss
  v
Fetch from Database --> Store in Redis --> Return data
```

## Basic Caching in Node.js

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

// Simulate a slow database query
async function getUserFromDatabase(userId) {
  // Pretend this takes 100ms
  return { id: userId, name: 'Alice', email: 'alice@example.com' };
}

async function getUser(userId) {
  const cacheKey = `user:${userId}`;

  // 1. Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    console.log('Cache hit!');
    return JSON.parse(cached);
  }

  // 2. Cache miss - fetch from database
  console.log('Cache miss - fetching from database');
  const user = await getUserFromDatabase(userId);

  // 3. Store in cache with 1-hour TTL
  await redis.setex(cacheKey, 3600, JSON.stringify(user));

  return user;
}

// Usage
const user = await getUser(42);
console.log(user);
```

## Basic Caching in Python

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_user(user_id: int) -> dict:
    cache_key = f"user:{user_id}"

    # Check cache
    cached = r.get(cache_key)
    if cached:
        print("Cache hit!")
        return json.loads(cached)

    # Cache miss
    print("Cache miss - querying database")
    user = query_user_from_db(user_id)  # Your DB query here

    # Store with 1-hour TTL
    r.setex(cache_key, 3600, json.dumps(user))

    return user

def query_user_from_db(user_id: int) -> dict:
    # Simulated DB query
    return {"id": user_id, "name": "Alice", "email": "alice@example.com"}
```

## Caching with a Generic Wrapper

```javascript
async function withCache(key, ttl, fetchFn) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const data = await fetchFn();
  await redis.setex(key, ttl, JSON.stringify(data));
  return data;
}

// Usage
const products = await withCache('products:featured', 300, async () => {
  return db.query('SELECT * FROM products WHERE featured = 1 LIMIT 10');
});

const config = await withCache('app:config', 60, async () => {
  return fetchConfigFromAPI();
});
```

## Cache Invalidation

When data changes, remove or update the cache:

```javascript
async function updateUser(userId, newData) {
  // Update the database
  await db.update('users', userId, newData);

  // Invalidate the cache
  await redis.del(`user:${userId}`);
}

async function updateUserAndCache(userId, newData) {
  // Update the database
  const updated = await db.update('users', userId, newData);

  // Update the cache directly (write-through)
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(updated));

  return updated;
}
```

## Checking TTL and Debugging

```bash
# Check if a key exists
EXISTS user:42

# Check remaining TTL (in seconds)
TTL user:42
# Returns -1 if no TTL, -2 if key doesn't exist

# Get the value
GET user:42

# Delete a key manually
DEL user:42

# List keys matching a pattern (avoid on large databases)
KEYS user:*
```

## Caching Lists and Objects

```javascript
// Cache a list of items
async function getFeaturedProducts() {
  const key = 'products:featured';
  const cached = await redis.get(key);

  if (cached) return JSON.parse(cached);

  const products = await db.query('SELECT * FROM products WHERE featured = 1');
  await redis.setex(key, 300, JSON.stringify(products)); // 5-minute cache
  return products;
}

// Cache with hash for individual field access
async function cacheUserProfile(userId, profile) {
  const key = `user:${userId}:profile`;
  await redis.hset(key, {
    name: profile.name,
    email: profile.email,
    avatar: profile.avatar
  });
  await redis.expire(key, 3600);
}
```

## Cache Hit Rate Monitoring

```bash
# Check hit/miss statistics
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# Calculate hit rate
# hit_rate = hits / (hits + misses)
```

```python
def get_cache_hit_rate():
    info = r.info('stats')
    hits = info['keyspace_hits']
    misses = info['keyspace_misses']
    total = hits + misses
    if total == 0:
        return 0
    return round((hits / total) * 100, 2)

print(f"Cache hit rate: {get_cache_hit_rate()}%")
```

## Summary

Redis as a cache reduces database load and speeds up your application by storing frequently accessed data in memory. The cache-aside pattern is the simplest starting point: check Redis first, fall back to the database on a miss, and store the result with a TTL. Cache invalidation on writes keeps data fresh. Monitor your cache hit rate - a healthy cache should achieve 80-95%+ hit rates for most workloads.
