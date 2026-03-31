# How to Implement Smart Cache Invalidation with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, Invalidation, Tag-Based, Strategy

Description: Learn how to implement smart cache invalidation with Redis using tag-based grouping so you can invalidate related keys atomically without scanning all keys.

---

Simple TTL-based invalidation is passive - entries expire on schedule regardless of whether the underlying data changed. Smart invalidation is event-driven: when data changes, you immediately invalidate exactly the affected cache entries. The challenge is doing this efficiently when one record might affect many cached views.

## Tag-Based Invalidation

Associate cache keys with tags. When a record changes, invalidate all keys tagged with that record's identifier.

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

TAG_PREFIX = "cachetag:"
CACHE_TTL = 300

def cache_set(cache_key: str, value: dict, tags: list[str], ttl: int = CACHE_TTL):
    """Store a value and record it under each tag."""
    pipeline = r.pipeline()
    pipeline.set(cache_key, json.dumps(value), ex=ttl)
    for tag in tags:
        tag_key = f"{TAG_PREFIX}{tag}"
        pipeline.sadd(tag_key, cache_key)
        pipeline.expire(tag_key, ttl * 2)  # tags live a bit longer
    pipeline.execute()

def cache_get(cache_key: str) -> dict | None:
    raw = r.get(cache_key)
    return json.loads(raw) if raw else None

def invalidate_by_tag(tag: str):
    """Delete all cache keys associated with a tag."""
    tag_key = f"{TAG_PREFIX}{tag}"
    keys = r.smembers(tag_key)
    if keys:
        pipeline = r.pipeline()
        pipeline.delete(*keys)
        pipeline.delete(tag_key)
        pipeline.execute()
        print(f"Invalidated {len(keys)} keys for tag '{tag}'")
    else:
        print(f"No cached keys for tag '{tag}'")
```

## Example: Product Caching with Tags

```python
# Cache a product listing page - tagged with product and category IDs
cache_set(
    "page:product-list:category:electronics",
    {"products": [{"id": "p1", "name": "Laptop"}]},
    tags=["category:electronics", "product:p1"]
)

# Cache a single product page
cache_set(
    "page:product:p1",
    {"id": "p1", "name": "Laptop", "price": 999},
    tags=["product:p1"]
)

# Cache a user-specific product view
cache_set(
    "page:product:p1:user:u42",
    {"id": "p1", "name": "Laptop", "wishlist": True},
    tags=["product:p1", "user:u42"]
)

# When product p1 is updated, invalidate everything tagged with it
invalidate_by_tag("product:p1")
# Deletes: page:product:p1, page:product-list:category:electronics, page:product:p1:user:u42
```

## Invalidating on Database Write

```python
def update_product(product_id: str, new_data: dict, db):
    # 1. Write to DB
    db.execute("UPDATE products SET data=%s WHERE id=%s", (json.dumps(new_data), product_id))

    # 2. Invalidate all caches that depend on this product
    invalidate_by_tag(f"product:{product_id}")

def update_category(category_id: str, new_data: dict, db):
    db.execute("UPDATE categories SET data=%s WHERE id=%s", (json.dumps(new_data), category_id))
    invalidate_by_tag(f"category:{category_id}")
```

## Inspecting Tags

```bash
# See all cache keys under a tag
redis-cli SMEMBERS cachetag:product:p1

# Count cached keys per tag
redis-cli SCARD cachetag:product:p1

# Check if a specific key is in a tag group
redis-cli SISMEMBER cachetag:product:p1 "page:product:p1"
```

## Batch Tag Invalidation

```python
def invalidate_tags(tags: list[str]):
    pipeline = r.pipeline()
    all_keys = set()
    for tag in tags:
        tag_key = f"{TAG_PREFIX}{tag}"
        keys = r.smembers(tag_key)
        all_keys.update(keys)
        pipeline.delete(tag_key)
    if all_keys:
        pipeline.delete(*all_keys)
    pipeline.execute()
```

## Summary

Smart cache invalidation with tag-based grouping lets you atomically remove all cache entries related to a changed record without key scanning. Store each cached key in Redis Sets indexed by tag, and on data change delete the tag set and all its member keys in a single pipeline. This replaces passive TTL expiry with precise, event-driven invalidation.

