# How to Build a Semantic Search Engine with Redis Vector Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector Search, Semantic Search, Embedding

Description: Build a semantic search engine using Redis Vector Search and sentence embeddings to find conceptually similar documents beyond keyword matching.

---

Semantic search finds documents based on meaning rather than exact keywords. Redis Vector Search (available via RedisSearch) stores and queries high-dimensional vector embeddings, enabling sub-millisecond similarity search at scale.

## Prerequisites

```bash
pip install redis sentence-transformers numpy
```

## Creating a Vector Index

Create a HNSW vector index for 384-dimensional sentence embeddings:

```bash
FT.CREATE doc_idx ON HASH PREFIX 1 doc:
  SCHEMA
    title TEXT
    body TEXT
    category TAG
    embedding VECTOR HNSW 6 TYPE FLOAT32 DIM 384 DISTANCE_METRIC COSINE
```

## Generating Embeddings

Use `sentence-transformers` to generate embeddings:

```python
import redis
import numpy as np
from sentence_transformers import SentenceTransformer

r = redis.Redis(host='localhost', port=6379, decode_responses=False)
model = SentenceTransformer('all-MiniLM-L6-v2')

def embed(text: str) -> bytes:
    vec = model.encode(text, normalize_embeddings=True)
    return vec.astype(np.float32).tobytes()

def add_document(doc_id: str, title: str, body: str, category: str):
    full_text = f"{title}. {body}"
    r.hset(f"doc:{doc_id}", mapping={
        "title": title,
        "body": body,
        "category": category,
        "embedding": embed(full_text)
    })

add_document("d001", "Redis Caching Strategies",
             "Learn TTL, LRU eviction, and write-through patterns.",
             "database")
add_document("d002", "Python Performance Tips",
             "Profiling, async IO, and memory optimization.",
             "programming")
add_document("d003", "Database Connection Pooling",
             "Manage database connections efficiently in production.",
             "database")
```

## Performing Semantic Search

Query using a vector embedding of the search phrase:

```python
def semantic_search(query: str, top_k: int = 5, category: str = None):
    query_vec = embed(query)

    if category:
        filter_expr = f"@category:{{{category}}}"
        search_query = f"({filter_expr})=>[KNN {top_k} @embedding $vec AS score]"
    else:
        search_query = f"*=>[KNN {top_k} @embedding $vec AS score]"

    result = r.execute_command(
        'FT.SEARCH', 'doc_idx', search_query,
        'PARAMS', 2, 'vec', query_vec,
        'RETURN', 3, 'title', 'category', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    count = result[0]
    docs = []
    for i in range(1, len(result), 2):
        doc_id = result[i].decode()
        fields = {}
        raw = result[i+1]
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        docs.append({"id": doc_id, **fields})
    return docs

results = semantic_search("how to speed up my app")
for doc in results:
    print(f"{doc['title']} - score: {doc['score']}")
```

## Hybrid Search: Semantic + Keyword

Combine vector similarity with a text keyword filter:

```python
def hybrid_search(query: str, keyword: str, top_k: int = 5):
    query_vec = embed(query)
    filter_expr = f"@body:({keyword})"
    search_query = f"({filter_expr})=>[KNN {top_k} @embedding $vec AS score]"

    return r.execute_command(
        'FT.SEARCH', 'doc_idx', search_query,
        'PARAMS', 2, 'vec', query_vec,
        'RETURN', 3, 'title', 'category', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )
```

## Summary

Redis Vector Search combines the speed of Redis with the power of semantic embeddings, enabling you to build a search engine that understands user intent rather than just matching keywords. By indexing documents as HNSW vectors and querying with sentence embeddings, you get near-millisecond semantic search at scale. Combine with TAG and TEXT filters for hybrid search that balances precision and recall.
