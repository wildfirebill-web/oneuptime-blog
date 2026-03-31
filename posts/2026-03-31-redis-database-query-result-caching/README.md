# How to Implement Database Query Result Caching with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Query Caching, Database, Performance, Cache-Aside, Python

Description: Implement database query result caching with Redis to serve repeated queries from memory, with consistent cache key generation, TTL strategy, and invalidation patterns.

---

## Why Cache Query Results

Database queries - even with optimal indexes - add latency and consume CPU on your database server. Caching query results in Redis serves repeated reads from memory in under 1ms, reduces database connections, and provides a buffer during traffic spikes.

## Cache Key Generation

A good cache key uniquely identifies a query and its parameters:

```python
import hashlib
import json

def make_cache_key(prefix: str, sql: str, params: tuple = ()) -> str:
    query_signature = json.dumps({"sql": sql, "params": list(params)}, sort_keys=True)
    hash_str = hashlib.sha256(query_signature.encode()).hexdigest()[:16]
    return f"{prefix}:{hash_str}"

# Examples
key1 = make_cache_key("db", "SELECT * FROM users WHERE status = %s", ("active",))
key2 = make_cache_key("db", "SELECT * FROM products WHERE category = %s AND price < %s", ("electronics", 100))
print(key1)  # db:a3f2b1c4d5e6f7a8
print(key2)  # db:b2c3d4e5f6a7b8c9
```

## Generic Query Cache Class

```python
import redis
import json
import psycopg2
from typing import Any, Optional

class QueryCache:
    def __init__(self, redis_client, db_connection, default_ttl: int = 300):
        self.r = redis_client
        self.db = db_connection
        self.default_ttl = default_ttl
        self._hits = 0
        self._misses = 0

    def query(self, sql: str, params: tuple = (), ttl: int = None, namespace: str = "db") -> list:
        cache_key = make_cache_key(namespace, sql, params)
        ttl = ttl if ttl is not None else self.default_ttl

        cached = self.r.get(cache_key)
        if cached is not None:
            self._hits += 1
            return json.loads(cached)

        self._misses += 1
        with self.db.cursor() as cursor:
            cursor.execute(sql, params)
            rows = cursor.fetchall()
            columns = [d[0] for d in cursor.description]
            result = [dict(zip(columns, row)) for row in rows]

        self.r.set(cache_key, json.dumps(result, default=str), ex=ttl)
        return result

    def invalidate(self, namespace: str = "db", pattern: str = "*"):
        keys = list(self.r.scan_iter(f"{namespace}:{pattern}"))
        if keys:
            self.r.delete(*keys)
            print(f"Invalidated {len(keys)} cache keys")

    def hit_rate(self) -> float:
        total = self._hits + self._misses
        return self._hits / total if total > 0 else 0.0

    def stats(self) -> dict:
        return {
            "hits": self._hits,
            "misses": self._misses,
            "hit_rate": f"{self.hit_rate() * 100:.1f}%"
        }

r = redis.Redis(decode_responses=True)
pg = psycopg2.connect(host="localhost", database="myapp", user="postgres", password="password")

cache = QueryCache(r, pg, default_ttl=300)

users = cache.query("SELECT id, name, email FROM users WHERE status = %s", ("active",))
print(f"Users: {len(users)}")
print(cache.stats())
```

## Namespace-Based Invalidation

Group related queries under a namespace for bulk invalidation:

```python
def get_users_by_role(cache: QueryCache, role: str):
    return cache.query(
        "SELECT id, name, email FROM users WHERE role = %s ORDER BY name",
        (role,),
        ttl=600,
        namespace="users"
    )

def update_user_role(cache: QueryCache, user_id: int, new_role: str):
    with pg.cursor() as cursor:
        cursor.execute("UPDATE users SET role = %s WHERE id = %s", (new_role, user_id))
    pg.commit()

    # Invalidate all user-namespace queries
    cache.invalidate(namespace="users")
    # Also invalidate specific user
    r.delete(f"user:{user_id}")
```

## TTL Strategy by Query Type

Different query types warrant different TTLs:

```python
TTL_STRATEGY = {
    # Frequently updated
    "stock_levels": 30,
    "active_sessions": 60,
    "recent_orders": 120,

    # Moderately updated
    "user_profiles": 300,
    "product_details": 600,

    # Rarely updated
    "categories": 3600,
    "country_list": 86400,
    "config_values": 3600,
}

def smart_query(sql: str, params: tuple, query_type: str):
    ttl = TTL_STRATEGY.get(query_type, 300)
    return cache.query(sql, params, ttl=ttl, namespace=query_type)
```

## Caching Aggregations and Reports

```python
def get_daily_sales_report(date_str: str):
    cache_key = f"report:daily_sales:{date_str}"

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    with pg.cursor() as cursor:
        cursor.execute("""
            SELECT
                DATE(created_at) as date,
                COUNT(*) as order_count,
                SUM(total_amount) as revenue
            FROM orders
            WHERE DATE(created_at) = %s
            GROUP BY DATE(created_at)
        """, (date_str,))
        row = cursor.fetchone()
        result = {"date": date_str, "order_count": row[0], "revenue": float(row[1] or 0)}

    # Past dates can be cached longer - they don't change
    from datetime import datetime, date
    is_today = datetime.strptime(date_str, "%Y-%m-%d").date() == date.today()
    ttl = 60 if is_today else 86400

    r.set(cache_key, json.dumps(result), ex=ttl)
    return result
```

## Preventing Cache Stampede

Use a lock to prevent multiple processes from simultaneously querying the database on a cache miss:

```python
def query_with_stampede_protection(sql: str, params: tuple, ttl: int = 300):
    cache_key = make_cache_key("db", sql, params)
    lock_key = f"lock:{cache_key}"

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Try to acquire lock - only one process queries the DB
    acquired = r.set(lock_key, "1", nx=True, ex=30)
    if not acquired:
        # Another process is fetching - wait briefly and retry cache
        import time
        time.sleep(0.1)
        cached = r.get(cache_key)
        return json.loads(cached) if cached else []

    try:
        with pg.cursor() as cursor:
            cursor.execute(sql, params)
            rows = cursor.fetchall()
            columns = [d[0] for d in cursor.description]
            result = [dict(zip(columns, row)) for row in rows]
        r.set(cache_key, json.dumps(result, default=str), ex=ttl)
        return result
    finally:
        r.delete(lock_key)
```

## Summary

Database query result caching with Redis uses a hash of the SQL query and parameters as the cache key to ensure uniqueness. A namespace-based key structure enables targeted bulk invalidation when underlying data changes. Different TTLs for different query types balance freshness and database load. Stampede protection with a distributed lock prevents thundering herd problems when a popular cache entry expires simultaneously for many clients.
