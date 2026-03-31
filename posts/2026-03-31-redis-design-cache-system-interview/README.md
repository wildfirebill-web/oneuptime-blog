# How to Design a Cache System Using Redis in a System Design Interview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, System Design, Cache, Interview, Architecture

Description: A structured approach to designing a Redis-backed cache system in a system design interview, covering patterns, eviction, and trade-offs.

---

Designing a cache system with Redis is one of the most common system design interview questions. Interviewers want to see that you understand not just how to use Redis, but how to design an entire caching layer with the right trade-offs. Here is a structured approach.

## Start with Requirements

Before designing, clarify:

- Read-heavy or write-heavy workload?
- What is the acceptable staleness (cache TTL)?
- What data must never be missing (critical vs. nice-to-have in cache)?
- Expected QPS and data size?

## Choose a Caching Pattern

**Cache-Aside (Lazy Loading)** - Most common:

```text
1. App checks Redis for key
2. Cache miss: fetch from DB, write to Redis, return data
3. Cache hit: return data from Redis directly
```

```python
def get_user(user_id):
    data = redis.get(f"user:{user_id}")
    if data:
        return json.loads(data)
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(f"user:{user_id}", 300, json.dumps(user))
    return user
```

**Write-Through** - Consistent but slower writes:

```text
Every DB write also updates the cache immediately.
No stale data, but adds latency to writes.
```

**Write-Behind (Write-Back)** - High throughput writes:

```text
Write to cache first, flush to DB asynchronously.
Fastest writes, risk of data loss if cache fails.
```

## Design the Key Schema

Keys should be structured for easy management:

```text
user:{user_id}           -> user profile hash
product:{product_id}     -> product details string (JSON)
session:{token}          -> session data with TTL
feed:{user_id}:page:{n}  -> paginated news feed
```

## Configure Eviction and Memory

```bash
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

For user-facing caches, `allkeys-lru` is usually the right choice. For session caches where you only want to evict expired sessions, use `volatile-ttl`.

## Handle Cache Invalidation

On data update, invalidate the cache entry:

```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id, data)
    redis.delete(f"user:{user_id}")
    # Next read will repopulate from DB
```

For related cached objects (e.g., user appears in multiple feed caches), consider tagging or a versioned key approach.

## Address Common Problems

**Cache Stampede:** Use a mutex lock to prevent many parallel DB reads on cache miss.

**Cache Penetration:** Cache negative results (e.g., `SET missing:user:999 "null" EX 60`) to prevent repeated DB hits for non-existent keys.

**Hot Keys:** Distribute a popular key across multiple shards by appending a random suffix and reading from any shard.

## Summary

A well-designed Redis cache system starts with choosing the right caching pattern (cache-aside for most use cases), a consistent key naming schema, and appropriate eviction policy. Discussing cache invalidation, stampede prevention, and hot key handling demonstrates the production-level thinking interviewers look for in system design rounds.
