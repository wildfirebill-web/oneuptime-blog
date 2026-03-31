# How to Use Native Vector Sets in Redis 8.0+

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector Sets, Redis 8, AI, Similarity Search

Description: Learn how to use Redis 8.0 native vector sets to store and search high-dimensional vectors for AI-powered similarity search without external modules.

---

## What Are Native Vector Sets

Redis 8.0 introduced vector sets as a first-class data type, eliminating the need for the RediSearch module to perform vector similarity search. Vector sets store high-dimensional floating-point vectors and support approximate nearest neighbor (ANN) search using the HNSW (Hierarchical Navigable Small World) algorithm.

This enables semantic search, recommendation engines, and embedding-based retrieval directly in Redis without external dependencies.

## Adding Vectors to a Vector Set

Use `VADD` to add a vector to a vector set:

```bash
redis-cli VADD products FP32 1.2 0.5 0.9 0.3 product:1001
redis-cli VADD products FP32 0.8 1.1 0.4 0.7 product:1002
redis-cli VADD products FP32 1.1 0.6 0.8 0.2 product:1003
```

The syntax is:

```text
VADD key [FP32 | FP64 | VALUES count] vector... element
```

- `FP32`: 32-bit float vectors (most common)
- `FP64`: 64-bit float vectors (higher precision, more memory)
- `VALUES count`: specify dimension count explicitly

For 1536-dimensional OpenAI embeddings:

```bash
redis-cli VADD embeddings FP32 $(python3 -c "import random; print(' '.join(str(round(random.uniform(-1,1),4)) for _ in range(1536)))") doc:1
```

## Searching for Similar Vectors

Use `VSIM` to find the K nearest neighbors:

```bash
redis-cli VSIM products FP32 1.2 0.5 0.9 0.3 COUNT 3
```

```text
1) "product:1001"
2) "0.9998"
3) "product:1003"
4) "0.9912"
5) "product:1002"
6) "0.8745"
```

The result pairs each element with its similarity score (1.0 = identical).

## Searching Using an Existing Element

Search for items similar to an element already in the set using `ELE`:

```bash
redis-cli VSIM products ELE product:1001 COUNT 5
```

This is useful for "more like this" recommendations.

## Adding Metadata to Vectors

Attach JSON attributes to each vector using `VSETATTR`:

```bash
redis-cli VSETATTR products product:1001 '{"name": "Wireless Headphones", "category": "Electronics", "price": 79.99}'
redis-cli VSETATTR products product:1002 '{"name": "Bluetooth Speaker", "category": "Electronics", "price": 49.99}'
```

Retrieve attributes:

```bash
redis-cli VGETATTR products product:1001
```

```text
"{\"name\": \"Wireless Headphones\", \"category\": \"Electronics\", \"price\": 79.99}"
```

## Vector Set Statistics

Get information about a vector set:

```bash
redis-cli VINFO products
```

```text
 1) "quant-type"
 2) "fp32"
 3) "vector-dim"
 4) (integer) 4
 5) "size"
 6) (integer) 3
 7) "max-level"
 8) (integer) 2
 9) "hnsw-m"
10) (integer) 16
```

Count elements:

```bash
redis-cli VCARD products
```

```text
(integer) 3
```

Get the dimension of vectors in the set:

```bash
redis-cli VDIM products
```

```text
(integer) 4
```

## Removing Vectors

Remove a specific element:

```bash
redis-cli VREM products product:1002
```

```text
(integer) 1
```

## Using Vector Sets with Python

```python
import redis
import numpy as np

r = redis.Redis(host='localhost', port=6379)

# Add vectors
embedding1 = np.random.rand(1536).astype(np.float32)
embedding2 = np.random.rand(1536).astype(np.float32)

r.execute_command('VADD', 'docs', 'FP32', *embedding1.tolist(), 'doc:1')
r.execute_command('VADD', 'docs', 'FP32', *embedding2.tolist(), 'doc:2')

# Search similar vectors
query = np.random.rand(1536).astype(np.float32)
results = r.execute_command('VSIM', 'docs', 'FP32', *query.tolist(), 'COUNT', 5)

# Parse results (pairs of element, score)
pairs = [(results[i], results[i+1]) for i in range(0, len(results), 2)]
for element, score in pairs:
    print(f"{element.decode()}: {score}")
```

## Practical Example: Semantic Search

Store document embeddings and search for similar content:

```python
from openai import OpenAI
import redis

openai_client = OpenAI()
r = redis.Redis(host='localhost', port=6379)

def embed(text):
    response = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

documents = [
    ("doc:1", "Redis is an in-memory data structure store"),
    ("doc:2", "PostgreSQL is a relational database"),
    ("doc:3", "Redis supports pub/sub messaging"),
]

# Index documents
for doc_id, text in documents:
    vector = embed(text)
    r.execute_command('VADD', 'search_index', 'FP32', *vector, doc_id)
    r.execute_command('VSETATTR', 'search_index', doc_id, f'{{"text": "{text}"}}')

# Search
query_text = "in-memory caching database"
query_vector = embed(query_text)
results = r.execute_command('VSIM', 'search_index', 'FP32', *query_vector, 'COUNT', 3)
```

## Quantization Options

For memory efficiency at scale, Redis 8 vector sets support quantization:

```bash
# INT8 quantization (4x less memory than FP32, slight accuracy loss)
redis-cli VADD products INT8 1.2 0.5 0.9 0.3 product:1001

# Binary quantization (32x less memory, suitable for high-dimensional vectors)
redis-cli VADD products BIN 1.2 0.5 0.9 0.3 product:1001
```

## Summary

Redis 8.0 native vector sets provide built-in HNSW-based approximate nearest neighbor search without requiring external modules. With commands like `VADD`, `VSIM`, `VSETATTR`, and `VINFO`, you can build semantic search and recommendation systems directly in Redis. Support for FP32, FP64, INT8, and binary quantization lets you trade off between accuracy and memory consumption based on your use case.
