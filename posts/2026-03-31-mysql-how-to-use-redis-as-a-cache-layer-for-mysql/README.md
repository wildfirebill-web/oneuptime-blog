# How to Use Redis as a Cache Layer for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Redis, Caching, Performance, Architecture

Description: Learn how to implement Redis as a cache layer in front of MySQL using cache-aside, write-through, and cache invalidation patterns with practical code examples.

---

## Why Use Redis in Front of MySQL

MySQL queries take milliseconds to seconds depending on complexity. Redis serves data from memory in microseconds. For frequently-read data that changes infrequently, caching in Redis dramatically reduces MySQL load and application latency.

Common use cases:
- User session data
- Product catalog pages
- Dashboard aggregations
- Rate-limiting counters
- Database query results

## Cache-Aside Pattern

The most common pattern - the application checks Redis first, falls back to MySQL on miss, then stores the result in Redis:

```python
import redis
import mysql.connector
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
db = mysql.connector.connect(host='localhost', user='root', password='pass', database='app')

def get_user(user_id: int) -> dict:
    cache_key = f"user:{user_id}"
    TTL_SECONDS = 300  # 5 minutes

    # 1. Check Redis cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. Cache miss - query MySQL
    cursor = db.cursor(dictionary=True)
    cursor.execute("SELECT id, name, email FROM users WHERE id = %s", (user_id,))
    user = cursor.fetchone()

    if user:
        # 3. Store in Redis with TTL
        r.setex(cache_key, TTL_SECONDS, json.dumps(user, default=str))

    return user
```

## Write-Through Pattern

On every write, update both MySQL and Redis simultaneously:

```python
def update_user(user_id: int, name: str, email: str):
    cursor = db.cursor()

    # 1. Update MySQL
    cursor.execute(
        "UPDATE users SET name = %s, email = %s WHERE id = %s",
        (name, email, user_id)
    )
    db.commit()

    # 2. Update Redis cache
    cache_key = f"user:{user_id}"
    user = {"id": user_id, "name": name, "email": email}
    r.setex(cache_key, 300, json.dumps(user))
```

## Cache Invalidation Pattern

On writes, delete the cache key and let the next read repopulate it:

```python
def update_product(product_id: int, data: dict):
    cursor = db.cursor()
    cursor.execute(
        "UPDATE products SET name = %s, price = %s WHERE id = %s",
        (data['name'], data['price'], product_id)
    )
    db.commit()

    # Invalidate cache - next read will repopulate from MySQL
    r.delete(f"product:{product_id}")
    r.delete(f"product_list:all")     # Invalidate list caches too
```

## Caching Expensive Aggregations

```python
def get_dashboard_stats() -> dict:
    cache_key = "dashboard:stats"
    TTL = 60  # Refresh every minute

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    cursor = db.cursor(dictionary=True)
    cursor.execute("""
        SELECT
          COUNT(*) AS total_orders,
          SUM(total) AS total_revenue,
          AVG(total) AS avg_order_value,
          COUNT(DISTINCT customer_id) AS unique_customers
        FROM orders
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    """)
    stats = cursor.fetchone()

    r.setex(cache_key, TTL, json.dumps(stats, default=str))
    return stats
```

## Caching Query Results with Hashed Keys

For parameterized queries, hash the query parameters to create a cache key:

```python
import hashlib

def execute_cached_query(sql: str, params: tuple, ttl: int = 300):
    key_data = f"{sql}:{params}"
    cache_key = f"query:{hashlib.md5(key_data.encode()).hexdigest()}"

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    cursor = db.cursor(dictionary=True)
    cursor.execute(sql, params)
    result = cursor.fetchall()

    r.setex(cache_key, ttl, json.dumps(result, default=str))
    return result
```

## Redis Pipeline for Bulk Cache Warming

Pre-warm cache after a restart or deployment:

```python
def warm_product_cache():
    cursor = db.cursor(dictionary=True)
    cursor.execute("SELECT id, name, price, category FROM products WHERE active = 1")
    products = cursor.fetchall()

    pipe = r.pipeline()
    for product in products:
        key = f"product:{product['id']}"
        pipe.setex(key, 3600, json.dumps(product, default=str))
    pipe.execute()

    print(f"Warmed cache for {len(products)} products")
```

## Monitoring Cache Hit Rate

```python
def get_cache_stats():
    info = r.info('stats')
    hits = info['keyspace_hits']
    misses = info['keyspace_misses']
    total = hits + misses

    hit_rate = (hits / total * 100) if total > 0 else 0
    print(f"Cache hit rate: {hit_rate:.1f}% ({hits} hits, {misses} misses)")
    return hit_rate
```

A healthy cache hit rate is above 80-90% for frequently-accessed data.

## Summary

Redis as a MySQL cache layer dramatically reduces database load for read-heavy workloads. The cache-aside pattern is the most common and flexible approach - check Redis first, fall back to MySQL on miss, and always set a TTL to prevent stale data. For write-heavy paths, use cache invalidation to ensure consistency and avoid serving stale data to users.
