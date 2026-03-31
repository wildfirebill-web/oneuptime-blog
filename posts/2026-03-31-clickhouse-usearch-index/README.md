# How to Use usearch Skip Index in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Vector Search, usearch, ANN Index, Embedding

Description: Learn how to use the usearch approximate nearest neighbor index in ClickHouse for fast, accurate vector similarity searches on high-dimensional embeddings.

---

## What is the usearch Index

`usearch` is a high-performance approximate nearest neighbor (ANN) index in ClickHouse backed by the USearch library. It uses HNSW (Hierarchical Navigable Small World) graphs, which generally provides better recall and faster queries than the older `annoy` index.

## Requirements

Enable the experimental usearch index:

```sql
SET allow_experimental_usearch_index = 1;
```

Or in `config.xml`:

```xml
<allow_experimental_usearch_index>1</allow_experimental_usearch_index>
```

## Create a Table with usearch Index

```sql
CREATE TABLE semantic_search (
    id UInt64,
    title String,
    embedding Array(Float32),
    INDEX idx_usearch embedding TYPE usearch('L2Distance', 'f32') GRANULARITY 1
) ENGINE = MergeTree()
ORDER BY id
SETTINGS allow_experimental_usearch_index = 1;
```

Parameters: `usearch(distance_function, scalar_type)`

- `distance_function` - `'L2Distance'`, `'cosineDistance'`, or `'dotProduct'`
- `scalar_type` - `'f32'` or `'f64'`

## Insert Embeddings

```python
import clickhouse_connect
import numpy as np

client = clickhouse_connect.get_client(host="localhost")

rows = [
    (i, f"Article {i}", np.random.rand(384).astype(np.float32).tolist())
    for i in range(50000)
]
client.insert("semantic_search", rows, column_names=["id", "title", "embedding"])
```

## Query Nearest Neighbors

```sql
SET allow_experimental_usearch_index = 1;

SELECT
    id,
    title,
    L2Distance(embedding, [0.1, 0.2, ...]) AS distance
FROM semantic_search
ORDER BY distance ASC
LIMIT 10;
```

## Cosine Similarity Search

```sql
SELECT
    id,
    title,
    cosineDistance(embedding, [0.1, 0.2, ...]) AS dist
FROM semantic_search
ORDER BY dist ASC
LIMIT 10;
```

## usearch vs annoy

```text
Feature          | usearch (HNSW)     | annoy
-----------------|--------------------|------------------
Algorithm        | HNSW graph         | Random trees
Recall accuracy  | Higher             | Lower
Build time       | Slower             | Faster
Query speed      | Faster at scale    | Comparable
Memory usage     | Higher             | Lower
Incremental add  | Better             | Poor
```

## HNSW Parameters

```sql
INDEX idx_usearch embedding TYPE usearch('cosineDistance', 'f32')
```

Additional HNSW parameters can be set via server configuration. The defaults work well for most use cases.

## Check Index Size

```sql
SELECT
    formatReadableSize(sum(secondary_indices_bytes_on_disk)) AS usearch_index_size
FROM system.parts
WHERE table = 'semantic_search' AND active = 1;
```

## Combine with Metadata Filters

```sql
WITH [0.1, 0.2, ...] AS query_vec
SELECT id, title, cosineDistance(embedding, query_vec) AS dist
FROM semantic_search
WHERE id > 1000
ORDER BY dist ASC
LIMIT 5;
```

Note: metadata filters are applied after ANN candidate retrieval - pre-filtering is approximate.

## Summary

The usearch index in ClickHouse provides high-recall approximate nearest neighbor search using HNSW graphs. It outperforms the annoy index in accuracy and query speed for large datasets. Use it for semantic search, recommendation engines, and embedding similarity at scale, choosing the distance function that matches your embedding normalization.
