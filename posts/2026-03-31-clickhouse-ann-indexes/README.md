# How to Use Approximate Nearest Neighbor (ANN) Indexes in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ANN Index, Vector Search, usearch, Annoy, Performance

Description: Learn how to create and use Approximate Nearest Neighbor (ANN) indexes in ClickHouse to speed up vector similarity searches on large datasets.

---

When searching through millions of embedding vectors, a brute-force scan becomes too slow for production workloads. ClickHouse supports Approximate Nearest Neighbor (ANN) indexes to accelerate vector similarity queries by orders of magnitude. This post walks through creating, materializing, and querying ANN indexes.

## How ANN Indexes Work

ANN indexes precompute a graph or tree structure over the embedding vectors so that nearest neighbor queries can skip the majority of rows. They trade a small amount of recall accuracy for significant speed gains - typically finding 95-99% of true nearest neighbors while scanning only a fraction of the data.

ClickHouse supports two ANN index types:

- **usearch** - based on the HNSW (Hierarchical Navigable Small World) algorithm. Fast and highly accurate.
- **annoy** - based on random projection trees. Lighter memory footprint.

## Creating a Table with ANN Index

```sql
CREATE TABLE document_embeddings
(
    doc_id    UInt64,
    content   String,
    embedding Array(Float32),
    INDEX emb_idx embedding TYPE usearch('cosineDistance') GRANULARITY 100000000
)
ENGINE = MergeTree()
ORDER BY doc_id;
```

The `GRANULARITY` value for ANN indexes should be set large enough to cover the entire dataset in a single index granule - `100000000` is a common choice.

## Adding an ANN Index to an Existing Table

```sql
ALTER TABLE document_embeddings
    ADD INDEX emb_idx embedding
    TYPE usearch('cosineDistance')
    GRANULARITY 100000000;

-- Build the index on existing data
ALTER TABLE document_embeddings MATERIALIZE INDEX emb_idx;
```

## Querying with the ANN Index

To trigger index usage, write the WHERE clause using the same distance function and threshold:

```sql
SELECT
    doc_id,
    content,
    cosineDistance(embedding, [0.1, 0.9, 0.4]) AS score
FROM document_embeddings
WHERE cosineDistance(embedding, [0.1, 0.9, 0.4]) < 0.3
ORDER BY score ASC
LIMIT 20
SETTINGS ann_index_select_query_params = '{"ef": 128}';
```

For `annoy` indexes, use the `k` parameter instead:

```sql
SETTINGS ann_index_select_query_params = '{"search_k": 100}';
```

## Choosing Between usearch and annoy

| Feature | usearch (HNSW) | annoy |
|---------|---------------|-------|
| Algorithm | HNSW graph | Random projection trees |
| Memory use | Higher | Lower |
| Build time | Slower | Faster |
| Query speed | Faster | Moderate |
| Recall | Very high (>99%) | Good (~95%) |

Use `usearch` for production workloads where recall matters. Use `annoy` when memory is constrained or for prototyping.

## Checking Index Usage

```sql
EXPLAIN indexes = 1
SELECT doc_id, cosineDistance(embedding, [0.1, 0.9, 0.4]) AS score
FROM document_embeddings
WHERE cosineDistance(embedding, [0.1, 0.9, 0.4]) < 0.3
ORDER BY score ASC
LIMIT 10;
```

Look for `ANN Index` in the output to confirm the index is being used.

## Verifying Index Status

```sql
SELECT
    table,
    name,
    type,
    data_compressed_bytes,
    data_uncompressed_bytes
FROM system.data_skipping_indices
WHERE table = 'document_embeddings';
```

## Summary

ANN indexes in ClickHouse (`usearch` and `annoy`) dramatically speed up vector similarity searches by avoiding full table scans. Create them using the `INDEX ... TYPE usearch(...)` syntax, materialize them on existing data, and trigger them via distance function conditions in the WHERE clause. Use `EXPLAIN indexes = 1` to verify index usage.
