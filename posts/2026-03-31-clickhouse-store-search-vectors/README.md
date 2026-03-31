# How to Store and Search Vectors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Vector Search, Embedding, ANN, Array

Description: Learn how to store vector embeddings in ClickHouse and perform approximate nearest neighbor search for AI-powered similarity queries.

---

## Vector Storage in ClickHouse

ClickHouse can store vector embeddings as Array(Float32) columns and perform similarity search using distance functions. Starting from ClickHouse 23.x, experimental ANN (Approximate Nearest Neighbor) indexes enable efficient vector search without scanning all rows.

## Storing Embeddings

```sql
CREATE TABLE document_embeddings (
    doc_id      UInt64,
    title       String,
    content     String,
    embedding   Array(Float32),
    created_at  DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY doc_id
SETTINGS index_granularity = 8192;
```

Insert embeddings (e.g., 1536-dimensional OpenAI embeddings):

```sql
INSERT INTO document_embeddings (doc_id, title, content, embedding)
VALUES (1, 'ClickHouse Basics', '...content...', [0.123, 0.456, ...]);
```

## Exact Nearest Neighbor Search (Brute Force)

For smaller datasets (under 100K rows), brute-force cosine similarity works well:

```sql
WITH [0.1, 0.2, 0.3, ...] AS query_vector
SELECT
    doc_id,
    title,
    cosineDistance(embedding, query_vector) AS distance
FROM document_embeddings
ORDER BY distance ASC
LIMIT 10;
```

Other distance functions:

```sql
-- L2 (Euclidean) distance
SELECT doc_id, L2Distance(embedding, query_vector) AS dist FROM document_embeddings ORDER BY dist LIMIT 10;

-- L1 (Manhattan) distance
SELECT doc_id, L1Distance(embedding, query_vector) AS dist FROM document_embeddings ORDER BY dist LIMIT 10;
```

## Approximate Nearest Neighbor Index (ANN)

For large-scale vector search, add an ANN index:

```sql
ALTER TABLE document_embeddings
    ADD INDEX ann_idx embedding TYPE annoy(100) GRANULARITY 1;

ALTER TABLE document_embeddings MATERIALIZE INDEX ann_idx;
```

Query using the ANN index:

```sql
WITH [0.1, 0.2, 0.3, ...] AS query_vector
SELECT doc_id, title, cosineDistance(embedding, query_vector) AS dist
FROM document_embeddings
ORDER BY dist ASC
LIMIT 10
SETTINGS ann_index_select_query_params = '{"ef":200}';
```

## Filtering Before Vector Search

Combine metadata filters with vector similarity:

```sql
WITH [0.1, 0.2, ...] AS query_vector
SELECT doc_id, title, cosineDistance(embedding, query_vector) AS dist
FROM document_embeddings
WHERE created_at >= now() - INTERVAL 30 DAY
  AND category = 'technology'
ORDER BY dist ASC
LIMIT 10;
```

## Batch Similarity Computation

Find similar documents across an entire dataset:

```sql
SELECT
    a.doc_id   AS doc_a,
    b.doc_id   AS doc_b,
    cosineDistance(a.embedding, b.embedding) AS similarity
FROM document_embeddings AS a
CROSS JOIN document_embeddings AS b
WHERE a.doc_id < b.doc_id
  AND cosineDistance(a.embedding, b.embedding) < 0.1
LIMIT 100;
```

## Hybrid Search: Combining Full-Text and Vector Search

```sql
WITH [0.1, 0.2, ...] AS query_vector
SELECT
    doc_id,
    title,
    cosineDistance(embedding, query_vector) AS vector_score,
    position(content, 'clickhouse') > 0 AS keyword_match,
    (1 - cosineDistance(embedding, query_vector)) * 0.7
        + (position(content, 'clickhouse') > 0 ? 0.3 : 0) AS hybrid_score
FROM document_embeddings
ORDER BY hybrid_score DESC
LIMIT 10;
```

## Summary

ClickHouse supports vector storage and similarity search via Array(Float32) columns and distance functions. For small datasets, brute-force cosine distance queries are sufficient. For production-scale vector search, use ANN indexes to enable approximate nearest neighbor lookups. Combine vector search with metadata filters and keyword matching for powerful hybrid retrieval pipelines.
