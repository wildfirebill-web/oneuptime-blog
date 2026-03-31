# How to Use RediSearch Full-Text Search in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, RediSearch, Full-Text Search, redis-py

Description: Build full-text search indexes in Redis with RediSearch and query them from Python using redis-py's search client, including field types and filters.

---

RediSearch is a Redis module that adds full-text search, numeric range filtering, and geo queries on top of Redis hashes and JSON documents. It enables Elasticsearch-like search without a separate service.

## Prerequisites

Start Redis Stack which includes RediSearch:

```bash
docker run -d -p 6379:6379 redis/redis-stack-server:latest
pip install redis
```

## Creating a Search Index

```python
import redis
from redis.commands.search.field import TextField, NumericField, TagField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Create an index over hashes with the prefix "article:"
try:
    r.ft("idx:articles").create_index(
        fields=[
            TextField("title", weight=5.0),
            TextField("body"),
            TagField("category"),
            NumericField("published_at", sortable=True),
            NumericField("views", sortable=True),
        ],
        definition=IndexDefinition(prefix=["article:"], index_type=IndexType.HASH),
    )
    print("Index created")
except Exception as e:
    if "Index already exists" in str(e):
        print("Index already exists, skipping")
    else:
        raise
```

## Indexing Documents

```python
articles = [
    {
        "title": "Getting Started with Redis",
        "body": "Redis is an in-memory data store used as a cache and message broker.",
        "category": "tutorial",
        "published_at": 1743000000,
        "views": 1500,
    },
    {
        "title": "Advanced Redis Patterns",
        "body": "Learn pub/sub, streams, and sorted sets for building scalable systems.",
        "category": "advanced",
        "published_at": 1743100000,
        "views": 890,
    },
    {
        "title": "Redis vs Memcached",
        "body": "A comparison of Redis and Memcached for caching use cases.",
        "category": "comparison",
        "published_at": 1743200000,
        "views": 2200,
    },
]

for i, article in enumerate(articles):
    r.hset(f"article:{i+1}", mapping=article)
```

## Basic Full-Text Search

```python
from redis.commands.search.query import Query

# Simple keyword search
results = r.ft("idx:articles").search("redis")
print(f"Found {results.total} results")
for doc in results.docs:
    print(f"  {doc.id}: {doc.title}")
```

## Filtered Search

```python
# Search with category tag filter
q = Query("redis").add_filter(TagField("category") == "tutorial")
results = r.ft("idx:articles").search(q)

# Numeric range filter
q = Query("*").add_filter(NumericField("views") >= 1000)
results = r.ft("idx:articles").search(q)

# Sort by views descending
q = Query("cache").sort_by("views", asc=False).paging(0, 10)
results = r.ft("idx:articles").search(q)
```

## Aggregation

```python
from redis.commands.search.aggregation import AggregateRequest
from redis.commands.search.reducers import count, avg

# Count articles per category
req = AggregateRequest("*").group_by("@category", count().alias("num_articles"))
agg = r.ft("idx:articles").aggregate(req)
for row in agg.rows:
    print(row)
```

## Index Info and Management

```python
info = r.ft("idx:articles").info()
print(f"Num docs: {info['num_docs']}")
print(f"Indexing failures: {info['hash_indexing_failures']}")

# Drop and recreate
r.ft("idx:articles").dropindex(delete_documents=False)
```

## Summary

RediSearch provides full-text indexing and querying on Redis hashes and JSON documents. Use `redis-py`'s `ft()` client to create indexes with `TextField`, `NumericField`, and `TagField`, populate data as hashes, and run queries with filters and sorting. This removes the need for a separate Elasticsearch cluster for many search use cases.
