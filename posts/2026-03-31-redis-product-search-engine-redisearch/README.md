# How to Build a Product Search Engine with RediSearch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Search, Product, E-Commerce

Description: Build a fast product search engine using RediSearch with full-text search, numeric filtering, and relevance ranking.

---

RediSearch turns Redis into a full-text search engine with sub-millisecond query times. For product catalogs, it supports text search, numeric range filters, tag filters, and custom relevance scoring - all in one query.

## Creating a Product Search Index

```python
import redis
from redis.commands.search.field import TextField, NumericField, TagField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_product_index():
    try:
        r.ft("idx:products").info()
        print("Index already exists")
        return
    except Exception:
        pass

    schema = (
        TextField("$.name", as_name="name", weight=5.0),
        TextField("$.description", as_name="description", weight=1.0),
        TextField("$.brand", as_name="brand", weight=2.0),
        NumericField("$.price", as_name="price"),
        NumericField("$.rating", as_name="rating"),
        NumericField("$.stock", as_name="stock"),
        TagField("$.category", as_name="category"),
        TagField("$.tags", as_name="tags"),
    )

    r.ft("idx:products").create_index(
        schema,
        definition=IndexDefinition(prefix=["product:"], index_type=IndexType.JSON),
    )
```

## Indexing Products

```python
import json

def index_product(product: dict):
    key = f"product:{product['id']}"
    r.json().set(key, "$", product)

# Example product
index_product({
    "id": "prod_001",
    "name": "Wireless Noise-Cancelling Headphones",
    "description": "Premium over-ear headphones with 30-hour battery life",
    "brand": "SoundMax",
    "price": 149.99,
    "rating": 4.5,
    "stock": 200,
    "category": "electronics",
    "tags": ["audio", "wireless", "noise-cancelling"],
})
```

## Full-Text Search

```python
from redis.commands.search.query import Query

def search_products(query_text: str, limit: int = 20) -> list:
    query = (
        Query(query_text)
        .paging(0, limit)
        .sort_by("rating", asc=False)
    )
    results = r.ft("idx:products").search(query)
    return [
        {"id": doc.id, "score": doc.score, **json.loads(doc.json)}
        for doc in results.docs
    ]
```

## Price and Category Filter

```python
def search_with_filters(text: str, min_price: float = 0,
                         max_price: float = 9999,
                         category: str = None) -> list:
    filter_expr = f"@price:[{min_price} {max_price}]"
    if category:
        filter_expr += f" @category:{{{category}}}"

    query_str = f"{text} {filter_expr}" if text else filter_expr
    results = r.ft("idx:products").search(Query(query_str).paging(0, 20))
    return [json.loads(doc.json) for doc in results.docs]
```

## In-Stock Filter

```python
def search_in_stock(text: str) -> list:
    query_str = f"{text} @stock:[1 +inf]"
    results = r.ft("idx:products").search(Query(query_str))
    return [json.loads(doc.json) for doc in results.docs]
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your product search API latency and alert when query times exceed your SLA.

```bash
redis-cli FT.INFO idx:products | grep num_docs
redis-cli FT.SEARCH idx:products "headphones" LIMIT 0 3
```

## Summary

RediSearch creates a full-text index over Redis JSON documents, enabling complex product queries with text, numeric range, and tag filters in a single command. Field weights boost name matches over description matches for more intuitive relevance. The index updates automatically when product Hashes or JSON documents are modified.
