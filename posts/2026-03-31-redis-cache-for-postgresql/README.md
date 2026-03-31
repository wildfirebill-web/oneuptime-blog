# How to Use Redis as a Cache for PostgreSQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PostgreSQL, Caching, Cache-Aside, Performance, Database

Description: Use Redis as a cache layer for PostgreSQL to reduce database load and query latency using the cache-aside pattern with TTL-based expiration and cache invalidation.

---

## Cache-Aside Pattern

The cache-aside (lazy loading) pattern is the most common approach:
1. Check Redis for the result
2. If a cache hit, return the cached value
3. If a cache miss, query PostgreSQL, store the result in Redis, then return it

## Setup

Install dependencies:

```bash
pip install redis psycopg2-binary
```

## Basic Cache-Aside Implementation

```python
import redis
import psycopg2
import json
import hashlib

r = redis.Redis(decode_responses=True)

pg = psycopg2.connect(
    host="localhost",
    database="myapp",
    user="postgres",
    password="password"
)

def query_with_cache(sql: str, params: tuple = (), ttl: int = 300):
    # Create a cache key from the query and parameters
    cache_key = "pg:query:" + hashlib.md5(f"{sql}{params}".encode()).hexdigest()

    # Check cache first
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss - query PostgreSQL
    with pg.cursor() as cursor:
        cursor.execute(sql, params)
        rows = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        result = [dict(zip(columns, row)) for row in rows]

    # Store in Redis
    r.set(cache_key, json.dumps(result, default=str), ex=ttl)

    return result

# Example usage
users = query_with_cache("SELECT id, name, email FROM users WHERE status = %s", ("active",))
print(f"Found {len(users)} users")
```

## Entity-Level Caching

Cache individual entities by primary key:

```python
def get_user(user_id: int):
    cache_key = f"user:{user_id}"

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    with pg.cursor() as cursor:
        cursor.execute("SELECT id, name, email, created_at FROM users WHERE id = %s", (user_id,))
        row = cursor.fetchone()
        if not row:
            return None
        user = dict(zip(["id", "name", "email", "created_at"], row))

    r.set(cache_key, json.dumps(user, default=str), ex=3600)
    return user

def update_user(user_id: int, name: str, email: str):
    with pg.cursor() as cursor:
        cursor.execute(
            "UPDATE users SET name = %s, email = %s WHERE id = %s",
            (name, email, user_id)
        )
    pg.commit()

    # Invalidate cache
    r.delete(f"user:{user_id}")

def create_user(name: str, email: str):
    with pg.cursor() as cursor:
        cursor.execute(
            "INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id",
            (name, email)
        )
        user_id = cursor.fetchone()[0]
    pg.commit()
    return user_id
```

## Write-Through Caching

Write to both Redis and PostgreSQL simultaneously:

```python
def create_user_write_through(name: str, email: str):
    with pg.cursor() as cursor:
        cursor.execute(
            "INSERT INTO users (name, email, status) VALUES (%s, %s, 'active') RETURNING id, created_at",
            (name, email)
        )
        user_id, created_at = cursor.fetchone()
    pg.commit()

    user = {"id": user_id, "name": name, "email": email, "created_at": str(created_at)}
    r.set(f"user:{user_id}", json.dumps(user), ex=3600)

    return user
```

## Caching List Queries

```python
def get_active_users(page: int = 1, page_size: int = 20):
    cache_key = f"users:active:page:{page}:size:{page_size}"

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    offset = (page - 1) * page_size
    with pg.cursor() as cursor:
        cursor.execute(
            "SELECT id, name, email FROM users WHERE status = 'active' ORDER BY id LIMIT %s OFFSET %s",
            (page_size, offset)
        )
        rows = cursor.fetchall()
        result = [{"id": r[0], "name": r[1], "email": r[2]} for r in rows]

    r.set(cache_key, json.dumps(result), ex=60)
    return result
```

## Cache Warming

Pre-populate the cache at startup for frequently accessed records:

```python
def warm_cache(user_ids: list):
    with pg.cursor() as cursor:
        cursor.execute(
            "SELECT id, name, email FROM users WHERE id = ANY(%s)",
            (user_ids,)
        )
        rows = cursor.fetchall()

    pipe = r.pipeline()
    for row in rows:
        user = {"id": row[0], "name": row[1], "email": row[2]}
        pipe.set(f"user:{row[0]}", json.dumps(user), ex=3600)
    pipe.execute()
    print(f"Cache warmed with {len(rows)} users")

warm_cache(list(range(1, 1001)))
```

## Cache Hit Rate Monitoring

```python
cache_hits = 0
cache_misses = 0

def get_user_monitored(user_id: int):
    global cache_hits, cache_misses

    cached = r.get(f"user:{user_id}")
    if cached:
        cache_hits += 1
        return json.loads(cached)

    cache_misses += 1
    # ... fetch from PostgreSQL

def get_hit_rate():
    total = cache_hits + cache_misses
    if total == 0:
        return 0
    return cache_hits / total * 100
```

## Summary

Redis serves as an efficient cache-aside layer for PostgreSQL by storing query results and entity data with a TTL, dramatically reducing database load for read-heavy workloads. Entity-level caching by primary key enables precise cache invalidation on update. Cache warming pre-populates Redis at startup for hot records, and monitoring hit rates guides TTL tuning to achieve optimal cache efficiency.
