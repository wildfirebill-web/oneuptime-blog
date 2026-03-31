# How to Cache Mobile API Responses with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Caching, Mobile, API, Performance

Description: Speed up mobile APIs by caching JSON responses in Redis with smart TTLs, cache invalidation, and per-user cache keys.

---

Mobile apps are latency-sensitive. Users notice a 300ms delay. Caching API responses in Redis can cut response times to under 5ms for frequently requested data like user profiles, product listings, and configuration payloads.

## Building a Cache Key

A good cache key captures everything that makes a response unique: the endpoint, user or device context, and any relevant query parameters.

```python
import hashlib
import json

def make_cache_key(endpoint: str, user_id: str, params: dict) -> str:
    param_hash = hashlib.md5(
        json.dumps(params, sort_keys=True).encode()
    ).hexdigest()[:8]
    return f"api:{endpoint}:{user_id}:{param_hash}"
```

## Cache-Aside Pattern

The cache-aside (lazy loading) pattern is the most common approach. Check the cache first; on a miss, fetch from the database and populate the cache:

```python
import redis
import json
from typing import Optional, Callable

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_cached_response(
    key: str,
    ttl: int,
    fetch_fn: Callable
) -> dict:
    cached = r.get(key)
    if cached:
        return json.loads(cached)

    data = fetch_fn()
    r.setex(key, ttl, json.dumps(data))
    return data
```

Usage in a Flask route:

```python
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.route("/api/v1/feed")
def get_feed():
    user_id = request.headers.get("X-User-ID")
    params = {"page": request.args.get("page", "1")}
    key = make_cache_key("feed", user_id, params)

    def fetch():
        return fetch_feed_from_db(user_id, int(params["page"]))

    data = get_cached_response(key, ttl=60, fetch_fn=fetch)
    return jsonify(data)
```

## Storing Compressed Responses

For large payloads like product catalogs, compress before caching to reduce memory usage:

```python
import zlib
import base64

def set_compressed(key: str, data: dict, ttl: int):
    raw = json.dumps(data).encode()
    compressed = base64.b64encode(zlib.compress(raw)).decode()
    r.setex(key, ttl, compressed)

def get_compressed(key: str) -> Optional[dict]:
    val = r.get(key)
    if not val:
        return None
    raw = zlib.decompress(base64.b64decode(val))
    return json.loads(raw)
```

## Invalidating User-Specific Cache

When a user updates their profile, invalidate all cache entries for that user using a key scan pattern:

```python
def invalidate_user_cache(user_id: str):
    pattern = f"api:*:{user_id}:*"
    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, match=pattern, count=100)
        if keys:
            r.delete(*keys)
        if cursor == 0:
            break
```

For high-throughput systems, maintain a set of user keys for O(1) invalidation:

```python
def track_user_key(user_id: str, cache_key: str, ttl: int):
    index_key = f"user_keys:{user_id}"
    r.sadd(index_key, cache_key)
    r.expire(index_key, ttl + 60)

def invalidate_user_cache_fast(user_id: str):
    index_key = f"user_keys:{user_id}"
    keys = r.smembers(index_key)
    if keys:
        r.delete(*keys, index_key)
```

## Setting Appropriate TTLs

Different mobile endpoints warrant different cache durations:

```python
TTL_CONFIG = {
    "feed": 60,           # 1 minute - frequently updated
    "profile": 300,       # 5 minutes - changes rarely
    "catalog": 3600,      # 1 hour - stable product data
    "config": 86400,      # 24 hours - app configuration
}
```

## Summary

Caching mobile API responses in Redis dramatically reduces backend load and improves perceived app speed. The cache-aside pattern with smart TTLs handles most use cases, while index-based invalidation ensures users always see fresh data after updates. Compression further reduces memory overhead for large JSON payloads.
