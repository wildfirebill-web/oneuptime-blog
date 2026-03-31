# How to Use Redis as a Cache for PostgreSQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PostgreSQL, Caching, Cache-Aside Pattern, Performance

Description: Learn how to use Redis as a cache layer in front of PostgreSQL using the cache-aside pattern, with cache invalidation and TTL management.

---

## Why Cache PostgreSQL with Redis

PostgreSQL is highly capable, but some queries are expensive:
- Queries on large tables with complex JOINs
- Frequently-read reference data (user profiles, product catalogs)
- Aggregation queries (counts, sums) that scan many rows
- Session data accessed on every request

Redis serves these from memory in under 1ms vs. 10-100ms+ for PostgreSQL queries.

## Cache-Aside Pattern

The cache-aside (lazy loading) pattern is the most common approach:

1. Application checks Redis first
2. If cache hit: return cached value
3. If cache miss: query PostgreSQL, store result in Redis with TTL, return value

```python
import redis
import psycopg2
import json
from typing import Optional

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
pg = psycopg2.connect("postgresql://user:password@localhost/mydb")

def get_user(user_id: int) -> Optional[dict]:
    cache_key = f"user:{user_id}"

    # 1. Check cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. Cache miss - query PostgreSQL
    with pg.cursor() as cur:
        cur.execute(
            "SELECT id, name, email, role, created_at FROM users WHERE id = %s",
            (user_id,)
        )
        row = cur.fetchone()

    if row is None:
        return None

    user = {
        'id': row[0], 'name': row[1], 'email': row[2],
        'role': row[3], 'created_at': str(row[4])
    }

    # 3. Store in cache with TTL
    r.setex(cache_key, 3600, json.dumps(user))
    return user
```

## Cache Invalidation on Updates

When the underlying data changes, invalidate the cache:

```python
def update_user(user_id: int, name: str, email: str) -> bool:
    with pg.cursor() as cur:
        cur.execute(
            "UPDATE users SET name = %s, email = %s, updated_at = NOW() WHERE id = %s",
            (name, email, user_id)
        )
    pg.commit()

    # Invalidate the cache
    r.delete(f"user:{user_id}")
    return True

def delete_user(user_id: int) -> bool:
    with pg.cursor() as cur:
        cur.execute("DELETE FROM users WHERE id = %s", (user_id,))
    pg.commit()

    r.delete(f"user:{user_id}")
    return True
```

## Caching Complex Query Results

Cache the result of expensive aggregation queries:

```python
def get_user_order_summary(user_id: int) -> dict:
    cache_key = f"user:{user_id}:order_summary"

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    with pg.cursor() as cur:
        cur.execute("""
            SELECT
                COUNT(*) as total_orders,
                SUM(total_amount) as total_spent,
                MAX(created_at) as last_order_date,
                AVG(total_amount) as avg_order_value
            FROM orders
            WHERE user_id = %s AND status != 'cancelled'
        """, (user_id,))
        row = cur.fetchone()

    summary = {
        'total_orders': row[0],
        'total_spent': float(row[1] or 0),
        'last_order_date': str(row[2]) if row[2] else None,
        'avg_order_value': float(row[3] or 0)
    }

    r.setex(cache_key, 300, json.dumps(summary))  # 5 minute TTL
    return summary
```

## Multi-Key Cache Population (Batch Loading)

Avoid N+1 queries by fetching and caching in batches:

```python
def get_users_bulk(user_ids: list) -> dict:
    if not user_ids:
        return {}

    # Check which users are cached
    pipe = r.pipeline()
    for uid in user_ids:
        pipe.get(f"user:{uid}")
    cached_results = pipe.execute()

    # Separate hits from misses
    result = {}
    missing_ids = []
    for uid, cached in zip(user_ids, cached_results):
        if cached:
            result[uid] = json.loads(cached)
        else:
            missing_ids.append(uid)

    if missing_ids:
        # Batch query PostgreSQL for missing users
        with pg.cursor() as cur:
            cur.execute(
                "SELECT id, name, email, role FROM users WHERE id = ANY(%s)",
                (missing_ids,)
            )
            rows = cur.fetchall()

        # Cache each fetched user
        pipe = r.pipeline()
        for row in rows:
            user = {'id': row[0], 'name': row[1], 'email': row[2], 'role': row[3]}
            result[row[0]] = user
            pipe.setex(f"user:{row[0]}", 3600, json.dumps(user))
        pipe.execute()

    return result
```

## Cache Repository Pattern in Python

```python
import redis
import psycopg2
import json
from typing import Optional, List
from contextlib import contextmanager

class UserRepository:
    def __init__(self, redis_client, pg_conn, ttl: int = 3600):
        self.cache = redis_client
        self.db = pg_conn
        self.ttl = ttl

    def _cache_key(self, user_id: int) -> str:
        return f"pg:users:{user_id}"

    def find_by_id(self, user_id: int) -> Optional[dict]:
        key = self._cache_key(user_id)
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        with self.db.cursor() as cur:
            cur.execute("SELECT id, name, email, role FROM users WHERE id = %s", (user_id,))
            row = cur.fetchone()

        if not row:
            # Cache negative result to prevent DB hammering
            self.cache.setex(key, 60, json.dumps(None))
            return None

        user = {'id': row[0], 'name': row[1], 'email': row[2], 'role': row[3]}
        self.cache.setex(key, self.ttl, json.dumps(user))
        return user

    def save(self, user: dict) -> dict:
        with self.db.cursor() as cur:
            if 'id' in user:
                cur.execute(
                    "UPDATE users SET name = %s, email = %s WHERE id = %s RETURNING id, name, email, role",
                    (user['name'], user['email'], user['id'])
                )
            else:
                cur.execute(
                    "INSERT INTO users (name, email, role) VALUES (%s, %s, %s) RETURNING id, name, email, role",
                    (user['name'], user['email'], user.get('role', 'user'))
                )
            row = cur.fetchone()
        self.db.commit()

        saved = {'id': row[0], 'name': row[1], 'email': row[2], 'role': row[3]}
        self.cache.setex(self._cache_key(saved['id']), self.ttl, json.dumps(saved))
        return saved

    def delete(self, user_id: int):
        with self.db.cursor() as cur:
            cur.execute("DELETE FROM users WHERE id = %s", (user_id,))
        self.db.commit()
        self.cache.delete(self._cache_key(user_id))
```

## Monitoring Cache Performance

```python
def get_cache_metrics(prefix: str = "user:") -> dict:
    info = r.info('stats')
    return {
        'hits': info.get('keyspace_hits', 0),
        'misses': info.get('keyspace_misses', 0),
        'hit_rate': info.get('keyspace_hits', 0) / max(
            info.get('keyspace_hits', 0) + info.get('keyspace_misses', 0), 1
        ),
        'cached_keys': sum(
            1 for _ in r.scan_iter(f"{prefix}*")
        )
    }

metrics = get_cache_metrics()
print(f"Cache hit rate: {metrics['hit_rate']:.1%}")
print(f"Cached users: {metrics['cached_keys']}")
```

## Summary

Using Redis as a cache for PostgreSQL with the cache-aside pattern reduces database load and response latency by serving frequently accessed data from memory. Implement cache invalidation on every update and delete operation to maintain consistency, use pipeline operations for batch cache reads to avoid N+1 Redis round-trips, and set appropriate TTLs based on how frequently the underlying data changes. Monitor keyspace_hits and keyspace_misses to track your effective hit rate and optimize TTL values accordingly.
