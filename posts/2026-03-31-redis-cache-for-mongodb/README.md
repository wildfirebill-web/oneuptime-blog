# How to Use Redis as a Cache for MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, MongoDB, Caching, Cache-Aside, Performance, NoSQL

Description: Use Redis as a caching layer for MongoDB to reduce database load and improve read latency using the cache-aside pattern with document-level and query-level caching.

---

## Why Cache MongoDB with Redis

MongoDB queries against large collections with complex filters can be slow even with indexes. Redis caching reduces MongoDB read load by serving hot data from memory at sub-millisecond latency.

## Setup

```bash
pip install redis pymongo
```

## Basic Cache-Aside Pattern

```python
import redis
import json
import hashlib
from pymongo import MongoClient
from bson import ObjectId

r = redis.Redis(decode_responses=True)
mongo = MongoClient("mongodb://localhost:27017")
db = mongo.myapp

def json_serializer(obj):
    if isinstance(obj, ObjectId):
        return str(obj)
    raise TypeError(f"Object of type {type(obj)} is not JSON serializable")

def get_document(collection_name: str, doc_id: str, ttl: int = 3600):
    cache_key = f"{collection_name}:{doc_id}"

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    collection = db[collection_name]
    doc = collection.find_one({"_id": ObjectId(doc_id)})
    if doc:
        r.set(cache_key, json.dumps(doc, default=json_serializer), ex=ttl)

    return doc

def update_document(collection_name: str, doc_id: str, update: dict):
    collection = db[collection_name]
    collection.update_one({"_id": ObjectId(doc_id)}, {"$set": update})

    # Invalidate cache
    r.delete(f"{collection_name}:{doc_id}")

# Usage
product = get_document("products", "60d5ecb74f6b1a2b3c4d5e6f")
print(product)
```

## Query Result Caching

```python
def query_with_cache(collection_name: str, query: dict, projection: dict = None, ttl: int = 300):
    # Generate cache key from collection + query + projection
    key_data = json.dumps({
        "collection": collection_name,
        "query": query,
        "projection": projection
    }, sort_keys=True)
    cache_key = "mongo:query:" + hashlib.md5(key_data.encode()).hexdigest()

    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    collection = db[collection_name]
    cursor = collection.find(query, projection)
    results = list(cursor)

    r.set(cache_key, json.dumps(results, default=json_serializer), ex=ttl)
    return results

# Cache product searches
electronics = query_with_cache(
    "products",
    {"category": "electronics", "in_stock": True},
    {"name": 1, "price": 1, "rating": 1}
)
print(f"Found {len(electronics)} products")
```

## Hash-Based Document Caching

For documents with many fields where you only need some, use Redis hashes:

```python
def cache_user_as_hash(user_id: str, ttl: int = 3600):
    cache_key = f"user:{user_id}"

    if r.exists(cache_key):
        return r.hgetall(cache_key)

    user = db.users.find_one({"_id": ObjectId(user_id)})
    if not user:
        return None

    user_data = {
        "name": user.get("name", ""),
        "email": user.get("email", ""),
        "role": user.get("role", "viewer"),
        "plan": user.get("plan", "free")
    }
    r.hset(cache_key, mapping=user_data)
    r.expire(cache_key, ttl)
    return user_data

def get_user_field(user_id: str, field: str):
    cache_key = f"user:{user_id}"
    cached = r.hget(cache_key, field)
    if cached:
        return cached

    user = db.users.find_one({"_id": ObjectId(user_id)}, {field: 1})
    return user.get(field) if user else None
```

## Invalidation on Write

```python
def upsert_product(product_id: str, data: dict):
    db.products.replace_one(
        {"_id": ObjectId(product_id)},
        data,
        upsert=True
    )

    # Invalidate document cache
    r.delete(f"products:{product_id}")

    # Invalidate all related query caches (use a tag-based pattern)
    for key in r.scan_iter("mongo:query:*"):
        r.delete(key)
```

For production, use tag-based invalidation instead of scanning all keys:

```python
def cache_query_with_tags(cache_key: str, result, tags: list, ttl: int = 300):
    r.set(cache_key, json.dumps(result, default=json_serializer), ex=ttl)
    for tag in tags:
        r.sadd(f"cache:tag:{tag}", cache_key)
        r.expire(f"cache:tag:{tag}", ttl + 60)

def invalidate_by_tag(tag: str):
    tag_key = f"cache:tag:{tag}"
    cache_keys = r.smembers(tag_key)
    if cache_keys:
        r.delete(*cache_keys)
    r.delete(tag_key)
```

## Cache Statistics

```python
def get_cache_stats():
    info = r.info("stats")
    keyspace_hits = info.get("keyspace_hits", 0)
    keyspace_misses = info.get("keyspace_misses", 0)
    total = keyspace_hits + keyspace_misses

    return {
        "hit_rate": keyspace_hits / total if total > 0 else 0,
        "hits": keyspace_hits,
        "misses": keyspace_misses,
        "total_keys": r.dbsize()
    }

print(get_cache_stats())
```

## Bulk Cache Population

```python
def warm_product_cache(limit: int = 500):
    products = list(db.products.find({}, limit=limit))
    pipe = r.pipeline()
    for product in products:
        pid = str(product["_id"])
        pipe.set(f"products:{pid}", json.dumps(product, default=json_serializer), ex=3600)
    pipe.execute()
    print(f"Warmed {len(products)} products into cache")
```

## Summary

Redis caches MongoDB queries at the document level (by `_id`) and query level (by a hash of the query filter). Cache-aside pattern checks Redis first and falls back to MongoDB on a miss, then stores the result. Tag-based invalidation groups related cache keys for efficient bulk invalidation when documents change. Cache warming pre-populates hot documents at startup to eliminate the cold-start penalty.
