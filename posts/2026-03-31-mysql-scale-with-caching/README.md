# How to Scale MySQL with Caching

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Caching, Redis, Performance, Scaling

Description: Learn how to scale MySQL by adding a caching layer using Redis to reduce database load, lower query latency, and handle traffic spikes.

---

## Why Caching Scales MySQL

Even with read replicas and connection pooling, some MySQL queries are expensive and run frequently - product catalog lookups, session data, computed aggregates. Running these against MySQL on every request wastes database cycles. A caching layer (Redis, Memcached) stores query results in memory and serves them without touching MySQL at all.

Caching is most effective for data that is:
- Read far more often than it is written
- Expensive to compute or query
- Tolerant of brief staleness (cache TTL)

## Cache-Aside Pattern with Redis

The cache-aside (lazy loading) pattern is the most common approach. The application checks the cache first, falls back to MySQL on a miss, and then populates the cache:

```python
import json
import redis
import mysql.connector

cache = redis.Redis(host='cache.example.com', port=6379, decode_responses=True)
db = mysql.connector.connect(host='db.example.com', user='app',
                              password='secret', database='mydb')

def get_product(product_id: int):
    cache_key = f"product:{product_id}"

    # Try cache first
    cached = cache.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss - query MySQL
    cursor = db.cursor(dictionary=True)
    cursor.execute("SELECT * FROM products WHERE id = %s", (product_id,))
    product = cursor.fetchone()

    if product:
        # Store in cache with 10-minute TTL
        cache.setex(cache_key, 600, json.dumps(product, default=str))

    return product
```

## Cache Invalidation on Write

When a product is updated, invalidate its cache entry so the next read fetches fresh data:

```python
def update_product(product_id: int, price: float):
    cursor = db.cursor()
    cursor.execute(
        "UPDATE products SET price = %s WHERE id = %s",
        (price, product_id)
    )
    db.commit()

    # Invalidate cached version
    cache.delete(f"product:{product_id}")
```

## Caching Aggregates

Expensive aggregate queries are prime caching candidates:

```python
def get_total_orders_today():
    cache_key = "metrics:orders_today"
    cached = cache.get(cache_key)
    if cached:
        return int(cached)

    cursor = db.cursor()
    cursor.execute(
        "SELECT COUNT(*) FROM orders WHERE DATE(created_at) = CURDATE()"
    )
    count = cursor.fetchone()[0]

    # Cache for 60 seconds - slight staleness is acceptable for dashboards
    cache.setex(cache_key, 60, count)
    return count
```

## Write-Through Caching

For data where staleness is unacceptable, use write-through: always update the cache when writing to MySQL:

```python
def set_user_session(user_id: int, session_data: dict):
    # Write to MySQL for durability
    cursor = db.cursor()
    cursor.execute(
        "INSERT INTO sessions (user_id, data, updated_at) VALUES (%s, %s, NOW()) "
        "ON DUPLICATE KEY UPDATE data = VALUES(data), updated_at = NOW()",
        (user_id, json.dumps(session_data))
    )
    db.commit()

    # Write to Redis immediately
    cache.setex(f"session:{user_id}", 3600, json.dumps(session_data))
```

## Monitoring Cache Effectiveness

Track cache hit rate in Redis:

```bash
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
```

A hit rate below 80% suggests the TTL is too short or the cache key space is too large for available memory.

Also monitor the reduction in MySQL query load:

```sql
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Com_select';
```

Compare `Com_select` before and after adding caching to quantify the reduction in database reads.

## Summary

Caching with Redis scales MySQL by intercepting frequent, expensive reads before they reach the database. Use the cache-aside pattern for most scenarios, invalidate cache entries on writes, and use write-through for session or user-specific data where freshness matters. Monitor Redis hit rates and MySQL `Com_select` to measure caching effectiveness.
