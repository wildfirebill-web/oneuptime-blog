# How to Use annoy Skip Index for Vector Search in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, annoy, Vector Search, Approximate Nearest Neighbor, Embedding

Description: Learn how to create and use the annoy approximate nearest neighbor skip index in ClickHouse for similarity search on vector embeddings.

---

ClickHouse supports approximate nearest neighbor (ANN) search through skip indexes. The `annoy` index uses the Annoy algorithm to find vectors nearest to a query vector. This enables similarity search on embedding columns without scanning every row.

## Prerequisites

The annoy index is available in ClickHouse 23.x+. Enable experimental ANN feature if needed:

```sql
SET allow_experimental_annoy_index = 1;
```

## Creating a Table with annoy Index

```sql
SET allow_experimental_annoy_index = 1;

CREATE TABLE embeddings (
    id       UInt64,
    doc_text String,
    embedding Array(Float32),
    INDEX idx_emb embedding TYPE annoy(100) GRANULARITY 1
) ENGINE = MergeTree()
ORDER BY id;
```

The parameter `100` sets the number of trees. More trees = higher recall but slower build and larger index. Typical values range from 50 to 200.

## Inserting Embeddings

```sql
INSERT INTO embeddings VALUES
    (1, 'machine learning basics', [0.12, 0.45, 0.33, 0.89]),
    (2, 'deep learning neural nets', [0.15, 0.42, 0.31, 0.91]),
    (3, 'database query optimization', [0.78, 0.22, 0.91, 0.11]);
```

In practice, embeddings come from models like OpenAI `text-embedding-3-small` (1536 dims) or local sentence transformers.

## Querying for Nearest Neighbors

```sql
SELECT id, doc_text,
    L2Distance(embedding, [0.13, 0.44, 0.32, 0.90]) AS dist
FROM embeddings
ORDER BY dist ASC
LIMIT 5
SETTINGS ann_index_select_query_params = '{"num_trees_to_use": 10}'
```

Distance functions supported with annoy:
- `L2Distance` (Euclidean)
- `cosineDistance`
- `dotProduct`

## Materialize Index on Existing Data

```sql
ALTER TABLE embeddings MATERIALIZE INDEX idx_emb;
```

## Checking Index Usage

```sql
EXPLAIN indexes=1
SELECT id, L2Distance(embedding, [0.13, 0.44, 0.32, 0.90]) AS dist
FROM embeddings ORDER BY dist LIMIT 5
```

## Limitations

- The annoy index in ClickHouse is approximate - recall is not 100%
- Queries must use `ORDER BY distance_func(...) LIMIT N` pattern
- Not suitable for exact nearest neighbor requirements
- Index rebuild is required after major data changes

## annoy vs usearch

```text
Index      Algorithm    Recall    Build Speed    Memory
------     ---------    ------    -----------    ------
annoy      Random trees Medium    Fast           Moderate
usearch    HNSW         Higher    Slower         Higher
```

Use `annoy` for faster builds and moderate recall. Use `usearch` when recall accuracy is more critical.

## Summary

The `annoy` skip index in ClickHouse enables approximate nearest neighbor search on vector embedding columns. Create the index with a suitable number of trees, insert embeddings, and query with `L2Distance` or `cosineDistance` in an `ORDER BY ... LIMIT` pattern. It is well suited for recommendation systems, semantic search, and similarity matching at scale.
