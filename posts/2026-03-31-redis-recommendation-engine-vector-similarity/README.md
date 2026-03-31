# How to Build a Recommendation Engine with Redis Vector Similarity

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector Search, Recommendation, Embedding

Description: Build a real-time product recommendation engine using Redis Vector Similarity search to find items similar to what users are currently viewing.

---

Traditional recommendation systems require complex batch processing pipelines. With Redis Vector Similarity search, you can compute item-to-item recommendations in real time by storing product embeddings and querying for the nearest neighbors.

## How It Works

Each product is represented as a vector embedding capturing its attributes. When a user views a product, you query Redis for the K nearest vectors - those are the recommendations.

## Installation

```bash
pip install redis sentence-transformers numpy
```

## Creating the Product Vector Index

```python
import redis
import numpy as np
from sentence_transformers import SentenceTransformer

r = redis.Redis(host='localhost', port=6379, decode_responses=False)
embedder = SentenceTransformer('all-MiniLM-L6-v2')

# Create HNSW index for fast approximate nearest neighbor search
r.execute_command(
    'FT.CREATE', 'product_vec_idx', 'ON', 'HASH',
    'PREFIX', '1', 'prod:',
    'SCHEMA',
    'name', 'TEXT',
    'category', 'TAG',
    'price', 'NUMERIC',
    'embedding', 'VECTOR', 'HNSW', '6',
    'TYPE', 'FLOAT32', 'DIM', '384',
    'DISTANCE_METRIC', 'COSINE'
)
```

## Building Product Embeddings

Embed products using their descriptions and attributes:

```python
def build_product_text(name: str, category: str,
                       description: str, tags: list) -> str:
    return f"{name}. Category: {category}. {description}. Tags: {', '.join(tags)}"

def add_product(pid: str, name: str, category: str,
                description: str, tags: list, price: float):
    text = build_product_text(name, category, description, tags)
    vec = embedder.encode(text, normalize_embeddings=True)

    r.hset(f"prod:{pid}", mapping={
        "name": name.encode(),
        "category": category.encode(),
        "price": str(price).encode(),
        "embedding": vec.astype(np.float32).tobytes()
    })

add_product("p001", "Wireless Noise-Cancelling Headphones",
            "electronics",
            "Premium over-ear headphones with 30hr battery life.",
            ["audio", "wireless", "noise-cancelling"], 299.99)
add_product("p002", "Bluetooth Earbuds",
            "electronics",
            "True wireless earbuds with active noise cancellation.",
            ["audio", "wireless", "earbuds"], 129.99)
add_product("p003", "Running Shoes",
            "footwear",
            "Lightweight shoes designed for long distance running.",
            ["sport", "running", "lightweight"], 89.99)
```

## Finding Similar Products

Retrieve the top-K most similar products:

```python
def get_recommendations(pid: str, top_k: int = 5,
                        same_category: bool = False):
    # Fetch the product's embedding
    product_vec = r.hget(f"prod:{pid}", "embedding")
    if not product_vec:
        return []

    if same_category:
        category = r.hget(f"prod:{pid}", "category").decode()
        query = f"(@category:{{{category}}})=>[KNN {top_k + 1} @embedding $vec AS score]"
    else:
        query = f"*=>[KNN {top_k + 1} @embedding $vec AS score]"

    result = r.execute_command(
        'FT.SEARCH', 'product_vec_idx', query,
        'PARAMS', 2, 'vec', product_vec,
        'RETURN', 3, 'name', 'category', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    recommendations = []
    for i in range(1, len(result), 2):
        doc_id = result[i].decode()
        if doc_id == f"prod:{pid}":
            continue  # exclude the query product itself
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        recommendations.append({"id": doc_id, **fields})

    return recommendations[:top_k]

recs = get_recommendations("p001")
for r_item in recs:
    print(f"{r_item['name']} - similarity: {1 - float(r_item['score']):.3f}")
```

## Caching Recommendations

Cache results for frequently viewed products:

```python
import json

r_plain = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cached_recommendations(pid: str, top_k: int = 5,
                            ttl: int = 300) -> list:
    cache_key = f"rec:cache:{pid}:{top_k}"
    cached = r_plain.get(cache_key)
    if cached:
        return json.loads(cached)

    recs = get_recommendations(pid, top_k)
    r_plain.setex(cache_key, ttl, json.dumps(recs))
    return recs
```

## Summary

Redis Vector Similarity search enables real-time item recommendations without batch jobs or complex ML pipelines. By embedding product descriptions and querying for nearest neighbors at request time, you can deliver fresh, personalized recommendations. Cache results for popular products to reduce computation while maintaining low latency at scale.
