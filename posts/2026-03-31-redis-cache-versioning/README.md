# How to Implement Cache Versioning in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, Versioning, Strategy, Performance

Description: Learn how to implement cache versioning in Redis to invalidate all cached data instantly without deleting individual keys.

---

Cache invalidation is famously hard. Tracking down every key that needs to be deleted when your data model changes is error-prone and slow. Cache versioning solves this by embedding a version number in every cache key. When you want to invalidate everything, you just increment the version - old keys become unreachable and expire naturally.

## The Core Idea

Instead of storing data under `user:42`, store it under `v3:user:42`. When you bump the version to `v4`, every `v3:*` key is effectively invisible without any SCAN or DEL operations.

## Store the Current Version in Redis

Keep the global cache version as a Redis string:

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_version() -> int:
    version = r.get('cache:version')
    if version is None:
        r.set('cache:version', 1)
        return 1
    return int(version)

def increment_version() -> int:
    return r.incr('cache:version')
```

## Build Versioned Cache Keys

Prefix every key with the current version number:

```python
def versioned_key(key: str) -> str:
    version = get_version()
    return f"v{version}:{key}"

def cache_set(key: str, value: dict, ttl_seconds: int = 300):
    vkey = versioned_key(key)
    r.set(vkey, json.dumps(value), ex=ttl_seconds)

def cache_get(key: str) -> dict | None:
    vkey = versioned_key(key)
    data = r.get(vkey)
    return json.loads(data) if data else None
```

## Use the Versioned Cache

```python
def get_user(user_id: int) -> dict:
    cache_key = f"user:{user_id}"
    cached = cache_get(cache_key)
    if cached:
        print("Cache hit")
        return cached

    # Simulate DB fetch
    user = {"id": user_id, "name": "Alice", "email": "alice@example.com"}
    cache_set(cache_key, user, ttl_seconds=600)
    return user

def invalidate_all():
    """Bump the version to invalidate all cached data instantly."""
    new_version = increment_version()
    print(f"Cache version bumped to {new_version}")
```

## Per-Entity Versioning

For finer control, maintain a version per entity type rather than globally:

```python
def entity_version(entity: str) -> int:
    key = f"cache:version:{entity}"
    version = r.get(key)
    if version is None:
        r.set(key, 1)
        return 1
    return int(version)

def invalidate_entity(entity: str):
    r.incr(f"cache:version:{entity}")

# Usage
def get_product(product_id: int) -> dict:
    version = entity_version("product")
    cache_key = f"v{version}:product:{product_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    product = {"id": product_id, "name": "Widget", "price": 9.99}
    r.set(cache_key, json.dumps(product), ex=300)
    return product

# Invalidate only product cache
invalidate_entity("product")
```

## Clean Up Old Keys

Old versioned keys still occupy memory until they expire. Set a reasonable TTL on all cached values so Redis evicts them automatically:

```bash
# Confirm keys with old version prefix have expired
redis-cli keys "v1:*"
# Should return empty after TTL expires
```

## Summary

Cache versioning replaces complex key tracking with a single integer. Incrementing the version instantly makes all old cache entries unreachable, and they expire on their own schedule without any SCAN-and-DEL operations. Use a global version for simple cases and per-entity versions when you want to invalidate specific data types independently.
