# How to Use Redis as a Cache for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, MySQL, Cache, Performance, Database

Description: Learn how to use Redis as a cache layer in front of MySQL to reduce query load and improve response times using the cache-aside pattern.

---

MySQL query latency can be unpredictable under load, especially for complex joins or aggregations. Redis as a cache layer stores query results and serves them without touching MySQL, reducing latency from milliseconds to microseconds.

## Cache-Aside Pattern

```text
Request --> Check Redis
  Cache hit?  --> Return result directly
  Cache miss? --> Query MySQL --> Store in Redis --> Return result
```

## Setup

```python
import redis
import mysql.connector
import json
import hashlib
from typing import Optional

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

db = mysql.connector.connect(
    host="localhost",
    user="app",
    password="secret",
    database="myapp"
)

CACHE_TTL = 300  # 5 minutes

def make_cache_key(query: str, params: tuple = ()) -> str:
    raw = f"{query}:{params}"
    return "query:" + hashlib.sha256(raw.encode()).hexdigest()[:16]
```

## Cached Query Execution

```python
def cached_query(query: str, params: tuple = (), ttl: int = CACHE_TTL) -> list:
    cache_key = make_cache_key(query, params)

    # Try cache first
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss - query MySQL
    cursor = db.cursor(dictionary=True)
    cursor.execute(query, params)
    results = cursor.fetchall()
    cursor.close()

    # Store in cache
    r.set(cache_key, json.dumps(results), ex=ttl)
    return results

# Example usage
users = cached_query("SELECT id, name, email FROM users WHERE status = %s", ("active",))
products = cached_query("SELECT id, name, price FROM products WHERE category_id = %s", (5,), ttl=3600)
```

## Caching Individual Records

```python
def get_user_by_id(user_id: int) -> Optional[dict]:
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    cursor = db.cursor(dictionary=True)
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    user = cursor.fetchone()
    cursor.close()

    if user:
        r.set(cache_key, json.dumps(user), ex=600)
    return user

def invalidate_user(user_id: int):
    r.delete(f"user:{user_id}")
```

## Write-Through: Update Cache on Write

```python
def update_user(user_id: int, data: dict):
    cursor = db.cursor()
    cursor.execute(
        "UPDATE users SET name=%s, email=%s WHERE id=%s",
        (data["name"], data["email"], user_id)
    )
    db.commit()
    cursor.close()

    # Update cache immediately
    cache_key = f"user:{user_id}"
    existing_raw = r.get(cache_key)
    if existing_raw:
        existing = json.loads(existing_raw)
        existing.update(data)
        r.set(cache_key, json.dumps(existing), ex=600)
    else:
        r.delete(cache_key)  # Invalidate to force fresh load
```

## Monitoring Cache Effectiveness

```bash
# MySQL query stats (before/after)
mysql -e "SHOW STATUS LIKE 'Com_select';"

# Redis hit/miss info
redis-cli INFO stats | grep keyspace
redis-cli INFO stats | grep hits
```

## Cache Warm-Up on Startup

```python
def warm_top_products():
    cursor = db.cursor(dictionary=True)
    cursor.execute("SELECT * FROM products ORDER BY views DESC LIMIT 1000")
    products = cursor.fetchall()
    cursor.close()

    pipeline = r.pipeline()
    for product in products:
        pipeline.set(f"product:{product['id']}", json.dumps(product), ex=3600)
    pipeline.execute()
    print(f"Warmed {len(products)} products")
```

## Summary

Redis as a MySQL cache follows the cache-aside pattern: check Redis first, query MySQL only on a miss, then populate the cache. Invalidate or update cache entries on writes to prevent stale reads. Cache warm-up on startup fills common queries so the first user requests are never cold misses.

