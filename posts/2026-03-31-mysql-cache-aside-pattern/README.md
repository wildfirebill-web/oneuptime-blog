# How to Implement Cache-Aside Pattern with MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Cache, Pattern

Description: Learn how to implement the cache-aside pattern with MySQL and Redis to reduce database load by letting the application manage cache population and invalidation.

---

The cache-aside pattern (also called lazy loading) places the application in control of cache management. The application checks the cache before querying the database, populates the cache on a miss, and invalidates or updates the cache when data changes. This contrasts with write-through and read-through patterns where the cache itself manages database interaction.

## How Cache-Aside Works

1. Application receives a read request.
2. Application checks the cache for the key.
3. **Cache hit**: return cached value directly.
4. **Cache miss**: query MySQL, store result in cache, return result.
5. On writes: update MySQL and delete (or update) the cache entry.

## Basic Implementation with Redis

```python
import redis
import mysql.connector
import json

r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)
db = mysql.connector.connect(host='localhost', database='myapp', user='app', password='secret')

def get_user(user_id):
    key = f'user:{user_id}'

    # Step 1: Check cache
    cached = r.get(key)
    if cached:
        return json.loads(cached)

    # Step 2: Cache miss - query MySQL
    cursor = db.cursor(dictionary=True)
    cursor.execute('SELECT id, name, email, created_at FROM users WHERE id = %s', (user_id,))
    user = cursor.fetchone()

    if user:
        # Step 3: Populate cache (TTL = 10 minutes)
        r.setex(key, 600, json.dumps(user, default=str))

    return user
```

## Write Path: Invalidate on Update

On write, update MySQL first, then delete the cache entry:

```python
def update_user(user_id, name, email):
    cursor = db.cursor()

    # Write to MySQL
    cursor.execute(
        'UPDATE users SET name=%s, email=%s, updated_at=NOW() WHERE id=%s',
        (name, email, user_id)
    )
    db.commit()

    # Invalidate cache (delete forces re-population on next read)
    r.delete(f'user:{user_id}')
```

Always delete the cache entry rather than updating it directly - this avoids race conditions where two writers could set inconsistent values.

## Caching List Queries

Cache the results of list queries using a composite key:

```python
def get_products_by_category(category_id, page=1, per_page=20):
    key = f'products:cat:{category_id}:page:{page}'

    cached = r.get(key)
    if cached:
        return json.loads(cached)

    cursor = db.cursor(dictionary=True)
    cursor.execute(
        'SELECT id, name, price FROM products WHERE category_id=%s LIMIT %s OFFSET %s',
        (category_id, per_page, (page - 1) * per_page)
    )
    products = cursor.fetchall()

    r.setex(key, 300, json.dumps(products, default=str))
    return products


def invalidate_category_cache(category_id):
    # Delete all page cache entries for this category
    pattern = f'products:cat:{category_id}:page:*'
    keys = r.keys(pattern)
    if keys:
        r.delete(*keys)
```

## Choosing TTL Values

TTL (time to live) balances cache freshness against hit rate:

```python
TTL_CONFIG = {
    'user': 600,         # 10 minutes - changes occasionally
    'product': 1800,     # 30 minutes - changes infrequently
    'category_list': 300, # 5 minutes - changes rarely
    'session': 86400,    # 24 hours - session data
}
```

Short TTLs reduce stale data risk but increase cache misses. Long TTLs improve hit rates but require careful invalidation on writes.

## When to Use Cache-Aside

Cache-aside is appropriate when:
- Read-heavy workloads where the same data is read frequently
- Cache failures must not break reads (the app falls back to MySQL)
- Different data has different TTL requirements

It is less suitable for write-heavy workloads or when strong consistency between cache and database is critical.

## Summary

The cache-aside pattern with MySQL uses Redis or Memcached as a fast read layer. The application checks the cache first, queries MySQL on a miss, and invalidates cache entries on writes. This reduces database load significantly for read-heavy applications while keeping the implementation simple and failure-resilient.
