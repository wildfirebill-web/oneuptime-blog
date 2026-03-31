# How to Store ML Model Embeddings in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Embedding, Machine Learning, Array, cosineDistance, Vector Store

Description: Store and query high-dimensional ML model embeddings in ClickHouse using Array(Float32) columns and distance functions for similarity retrieval.

---

ML model embeddings - dense float vectors produced by models like BERT, OpenAI, or Sentence Transformers - need a storage backend that handles high-dimensional arrays efficiently. ClickHouse offers compact columnar storage for `Array(Float32)`, fast distance computations, and the ability to join embedding retrieval with structured analytical queries.

## Schema for Embedding Storage

```sql
CREATE TABLE item_embeddings
(
    item_id      UInt64,
    item_type    LowCardinality(String),
    model_name   LowCardinality(String),
    model_ver    String,
    embedding    Array(Float32),
    indexed_at   DateTime DEFAULT now()
)
ENGINE = ReplacingMergeTree(indexed_at)
ORDER BY (item_type, item_id, model_name);
```

Using ReplacingMergeTree lets you update embeddings when a model is retrained.

## Inserting Embeddings from Python

```python
import clickhouse_connect
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')
client = clickhouse_connect.get_client(host='localhost')

texts = ['ClickHouse is fast', 'Vector search at scale']
embeddings = model.encode(texts).tolist()

rows = [(i, 'text', 'all-MiniLM-L6-v2', 'v1', emb)
        for i, emb in enumerate(embeddings, start=1)]
client.insert('item_embeddings', rows,
              column_names=['item_id','item_type','model_name','model_ver','embedding'])
```

## Querying Nearest Neighbors

```sql
WITH [/* your query embedding, 384 floats */] AS q
SELECT
    item_id,
    cosineDistance(embedding, q) AS dist
FROM item_embeddings FINAL
WHERE model_name = 'all-MiniLM-L6-v2'
ORDER BY dist ASC
LIMIT 10;
```

## Filtering by Metadata Before Computing Distance

Pre-filtering on indexed columns reduces the number of distance computations:

```sql
WITH [/* query embedding */] AS q
SELECT item_id, dist
FROM (
    SELECT item_id, cosineDistance(embedding, q) AS dist
    FROM item_embeddings FINAL
    WHERE item_type = 'product'
      AND model_name = 'all-MiniLM-L6-v2'
      AND indexed_at >= today() - 7
) ORDER BY dist ASC LIMIT 20;
```

## Handling Model Updates

When you retrain a model, insert new embeddings with the same primary key. ReplacingMergeTree keeps only the latest version:

```sql
-- Re-embed item 1 with new model version
INSERT INTO item_embeddings VALUES
(1, 'text', 'all-MiniLM-L6-v2', 'v2', [/* new embedding */], now());
```

Query with `FINAL` to get the latest version immediately.

## Checking Dimension Consistency

```sql
SELECT model_name, model_ver, min(length(embedding)), max(length(embedding))
FROM item_embeddings FINAL
GROUP BY model_name, model_ver;
```

Min and max should be equal. A mismatch indicates a broken embedding pipeline.

## Summary

ClickHouse is a practical embedding store when you need to combine vector retrieval with structured filtering, aggregation, or joins. Use `Array(Float32)` with ReplacingMergeTree to support model updates, apply metadata pre-filters before computing distance, and verify dimension consistency across model versions.
