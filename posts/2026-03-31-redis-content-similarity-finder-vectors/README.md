# How to Build a Content Similarity Finder with Redis Vectors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector Search, Content, Similarity

Description: Find similar articles, blog posts, or documents in real time using Redis Vector Search and sentence embeddings to power content recommendations.

---

Content similarity powers "related articles," duplicate detection, and content clustering. Redis Vector Search makes it practical to build a similarity finder that operates in milliseconds, even across hundreds of thousands of documents.

## How It Works

1. Embed each document using a sentence transformer
2. Store embeddings in a Redis HNSW vector index
3. At query time, embed the target document and search for nearest neighbors

## Installation

```bash
pip install redis sentence-transformers numpy
```

## Creating the Index

```python
import redis
import numpy as np
from sentence_transformers import SentenceTransformer

r = redis.Redis(host='localhost', port=6379, decode_responses=False)
embedder = SentenceTransformer('all-MiniLM-L6-v2')

r.execute_command(
    'FT.CREATE', 'content_idx', 'ON', 'HASH',
    'PREFIX', '1', 'content:',
    'SCHEMA',
    'title', 'TEXT',
    'topic', 'TAG',
    'author', 'TAG',
    'embedding', 'VECTOR', 'HNSW', '6',
    'TYPE', 'FLOAT32', 'DIM', '384',
    'DISTANCE_METRIC', 'COSINE'
)
```

## Indexing Documents

Embed and store content pieces:

```python
def embed_document(title: str, body: str) -> bytes:
    text = f"{title}. {body[:500]}"  # use first 500 chars for speed
    vec = embedder.encode(text, normalize_embeddings=True)
    return vec.astype(np.float32).tobytes()

def index_content(content_id: str, title: str, body: str,
                  topic: str, author: str):
    r.hset(f"content:{content_id}", mapping={
        "title": title.encode(),
        "topic": topic.encode(),
        "author": author.encode(),
        "embedding": embed_document(title, body)
    })

index_content("a001", "Getting Started with Redis",
              "Redis is an in-memory data store used for caching...",
              "database", "alice")
index_content("a002", "Redis vs Memcached",
              "Both Redis and Memcached are in-memory stores, but...",
              "database", "bob")
index_content("a003", "Python Async Programming",
              "Asyncio enables concurrent IO-bound tasks in Python...",
              "programming", "alice")
```

## Finding Similar Content

```python
def find_similar(content_id: str, top_k: int = 5,
                 same_topic: bool = False) -> list:
    stored_vec = r.hget(f"content:{content_id}", "embedding")
    if not stored_vec:
        return []

    if same_topic:
        topic = r.hget(f"content:{content_id}", "topic").decode()
        query = (
            f"(@topic:{{{topic}}})"
            f"=>[KNN {top_k + 1} @embedding $vec AS score]"
        )
    else:
        query = f"*=>[KNN {top_k + 1} @embedding $vec AS score]"

    result = r.execute_command(
        'FT.SEARCH', 'content_idx', query,
        'PARAMS', 2, 'vec', stored_vec,
        'RETURN', 3, 'title', 'topic', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    results = []
    for i in range(1, len(result), 2):
        doc_id = result[i].decode()
        if doc_id == f"content:{content_id}":
            continue
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        similarity = round(1 - float(fields.get('score', 1.0)), 4)
        results.append({"id": doc_id,
                        "title": fields.get('title', ''),
                        "similarity": similarity})

    return results[:top_k]

similar = find_similar("a001", top_k=3)
for item in similar:
    print(f"{item['title']} - {item['similarity']:.4f}")
```

## Querying by Raw Text

Find similar content for an arbitrary text snippet (useful for new drafts):

```python
def find_similar_to_text(text: str, top_k: int = 5,
                          topic: str = None) -> list:
    query_vec = embedder.encode(text, normalize_embeddings=True)
    query_bytes = query_vec.astype(np.float32).tobytes()

    if topic:
        query = (
            f"(@topic:{{{topic}}})"
            f"=>[KNN {top_k} @embedding $vec AS score]"
        )
    else:
        query = f"*=>[KNN {top_k} @embedding $vec AS score]"

    result = r.execute_command(
        'FT.SEARCH', 'content_idx', query,
        'PARAMS', 2, 'vec', query_bytes,
        'RETURN', 2, 'title', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    results = []
    for i in range(1, len(result), 2):
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        results.append({"title": fields.get('title', ''),
                        "similarity": round(1 - float(fields.get('score', 1.0)), 4)})
    return results
```

## Summary

Redis Vector Search makes content similarity practical for production use. By embedding documents at index time and querying with HNSW approximate nearest neighbor search, you get sub-millisecond "related content" recommendations. Use the `same_topic` filter to keep recommendations topically relevant, and the raw text query to check for near-duplicates before publishing new content.
