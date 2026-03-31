# How to Use usearch Skip Index in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, usearch, Vector Search, HNSW, Approximate Nearest Neighbor, Embedding

Description: Learn how to create and use the usearch HNSW-based skip index in ClickHouse for high-recall approximate nearest neighbor search on vector embeddings.

---

The `usearch` index in ClickHouse uses the HNSW (Hierarchical Navigable Small World) algorithm from the USearch library to build an approximate nearest neighbor index on Array(Float32) columns. It offers higher recall than the annoy index at the cost of more memory during index construction.

## Enabling usearch

```sql
SET allow_experimental_usearch_index = 1;
```

## Creating a Table with usearch Index

```sql
SET allow_experimental_usearch_index = 1;

CREATE TABLE doc_embeddings (
    id        UInt64,
    content   String,
    embedding Array(Float32),
    INDEX idx_vec embedding
        TYPE usearch('L2Distance', 'f32')
        GRANULARITY 1
) ENGINE = MergeTree()
ORDER BY id;
```

Parameters:
- First argument: distance metric (`'L2Distance'`, `'cosineDistance'`)
- Second argument: scalar type (`'f32'` for Float32, `'f64'` for Float64)

## Inserting Embeddings

```sql
INSERT INTO doc_embeddings VALUES
    (1, 'clickhouse storage engine', [0.25, 0.60, 0.10, 0.05]),
    (2, 'vector similarity search', [0.22, 0.65, 0.08, 0.05]),
    (3, 'time series analytics', [0.80, 0.15, 0.45, 0.60]);
```

## Querying Nearest Neighbors

```sql
SELECT
    id,
    content,
    L2Distance(embedding, [0.23, 0.62, 0.09, 0.05]) AS dist
FROM doc_embeddings
ORDER BY dist ASC
LIMIT 10
```

For cosine similarity:

```sql
SELECT
    id,
    content,
    cosineDistance(embedding, [0.23, 0.62, 0.09, 0.05]) AS dist
FROM doc_embeddings
ORDER BY dist ASC
LIMIT 10
```

## Materialize Index on Existing Data

```sql
ALTER TABLE doc_embeddings MATERIALIZE INDEX idx_vec;
```

## Inspecting Index Size

```sql
SELECT formatReadableSize(sum(secondary_indices_compressed_bytes)) AS index_size
FROM system.parts
WHERE table = 'doc_embeddings' AND active = 1
```

## HNSW Tuning Parameters

HNSW has two main parameters that affect the quality/speed trade-off:

```text
Parameter         Default    Effect
---------         -------    ------
m                 16         Max connections per node; higher = better recall
ef_construction   128        Search depth during build; higher = better recall
```

These can be set in the index parameters when ClickHouse exposes them through settings.

## usearch vs annoy

```text
Index      Algorithm    Recall    Memory    Build Speed
------     ---------    ------    ------    -----------
usearch    HNSW         High      Higher    Moderate
annoy      Random trees Medium    Lower     Fast
```

Choose `usearch` when recall quality matters (semantic search, recommendations). Choose `annoy` when build speed and lower memory usage are priorities.

## Summary

The `usearch` index brings HNSW-based approximate nearest neighbor search to ClickHouse. Use it with Float32 embedding columns for semantic search, recommendation engines, and similarity matching. Query with `L2Distance` or `cosineDistance` in an `ORDER BY ... LIMIT` pattern, and materialize the index on existing data after creation.
