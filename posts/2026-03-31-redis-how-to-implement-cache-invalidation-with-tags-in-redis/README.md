# How to Implement Cache Invalidation with Tags in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Caching, Cache Invalidation, Tags, Backend

Description: Learn how to implement tag-based cache invalidation in Redis to efficiently expire groups of related cache entries without manual tracking.

---

## What Is Tag-Based Cache Invalidation

Tag-based cache invalidation lets you associate cache entries with one or more logical tags and then invalidate all entries sharing a tag at once. Instead of tracking individual keys, you group them by tag so a single command can bust an entire category of cached data.

This is especially useful when a single database record change affects many cached views, for example when a product is updated and you need to clear every page that displayed that product.

## Storing Tags with Cache Entries

The simplest pattern uses Redis Sets to track which keys belong to a given tag. When you write a cache entry, you also add the key to every tag set.

```bash
# Cache a product page and tag it with "product:42" and "category:electronics"
SET cache:product_page:42 "<html>..." EX 3600
SADD tag:product:42 cache:product_page:42
SADD tag:category:electronics cache:product_page:42
```

To add multiple tags in one round trip, use pipelining:

```python
import redis

r = redis.Redis()

def set_with_tags(key: str, value: str, tags: list, ttl: int = 3600):
    pipe = r.pipeline()
    pipe.set(key, value, ex=ttl)
    for tag in tags:
        pipe.sadd(f"tag:{tag}", key)
    pipe.execute()

set_with_tags(
    "cache:product_page:42",
    "<html>Product 42</html>",
    tags=["product:42", "category:electronics"],
    ttl=3600
)
```

## Invalidating by Tag

When the underlying data changes, fetch all keys in the tag set and delete them together with the tag set itself.

```python
def invalidate_tag(tag: str):
    tag_key = f"tag:{tag}"
    keys = r.smembers(tag_key)
    if keys:
        pipe = r.pipeline()
        for k in keys:
            pipe.delete(k)
        pipe.delete(tag_key)
        pipe.execute()

# When product 42 is updated
invalidate_tag("product:42")
```

Using a pipeline keeps the operation atomic at the client level and reduces round trips.

## Handling Tag Set Expiry

Tag sets themselves do not expire automatically. If the cache key expires before you explicitly invalidate the tag, the tag set will contain stale references. Clean up orphaned entries periodically:

```python
def clean_tag(tag: str):
    tag_key = f"tag:{tag}"
    members = r.smembers(tag_key)
    stale = [k for k in members if not r.exists(k)]
    if stale:
        r.srem(tag_key, *stale)
```

Alternatively, assign the same TTL to the tag set as to the longest-lived cache key under it, and refresh the TTL on every write:

```python
def set_with_tags(key: str, value: str, tags: list, ttl: int = 3600):
    pipe = r.pipeline()
    pipe.set(key, value, ex=ttl)
    for tag in tags:
        tag_key = f"tag:{tag}"
        pipe.sadd(tag_key, key)
        pipe.expire(tag_key, ttl)
    pipe.execute()
```

## Using Lua for Atomic Multi-Key Invalidation

For strict atomicity, wrap invalidation in a Lua script so no key can be written between the SMEMBERS read and the DEL calls:

```lua
-- invalidate_tag.lua
local tag_key = KEYS[1]
local members = redis.call("SMEMBERS", tag_key)
for _, k in ipairs(members) do
    redis.call("DEL", k)
end
redis.call("DEL", tag_key)
return #members
```

```python
invalidate_script = r.register_script(open("invalidate_tag.lua").read())

def invalidate_tag_atomic(tag: str):
    deleted = invalidate_script(keys=[f"tag:{tag}"])
    return deleted
```

## Practical Example - Product Catalog

```python
import redis

r = redis.Redis(decode_responses=True)

# Cache product detail page
def cache_product_page(product_id: int, html: str, category: str):
    key = f"cache:product:{product_id}"
    set_with_tags(key, html, tags=[
        f"product:{product_id}",
        f"category:{category}",
        "all_products"
    ], ttl=7200)

# Cache category listing
def cache_category_page(category: str, html: str):
    key = f"cache:category:{category}"
    set_with_tags(key, html, tags=[f"category:{category}"], ttl=3600)

# On product update
def on_product_update(product_id: int):
    invalidate_tag(f"product:{product_id}")

# On bulk restock
def on_bulk_restock():
    invalidate_tag("all_products")
```

## Summary

Tag-based cache invalidation in Redis uses Sets to map logical tags to cache keys, enabling bulk invalidation with a single call. Pairing pipelined writes with optional Lua-based atomic invalidation gives you efficient, consistent cache management at scale. Regular cleanup of stale tag entries prevents unbounded memory growth.
