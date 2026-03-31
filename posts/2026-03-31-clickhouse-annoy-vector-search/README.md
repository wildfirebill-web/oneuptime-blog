# How to Use annoy Skip Index for Vector Search in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Vector Search, annoy, ANN Index, Embedding

Description: Learn how to use the annoy approximate nearest neighbor skip index in ClickHouse to perform fast vector similarity searches on embedding columns.

---

## What is the annoy Index

`annoy` (Approximate Nearest Neighbors Oh Yeah) is an approximate nearest neighbor (ANN) index in ClickHouse that uses a forest of random projection trees to find nearest neighbors for vector data. It enables fast similarity search on embedding columns without scanning every row.

## Requirements

The annoy index is available in ClickHouse 23.x and later. Enable it in configuration:

```xml
<allow_experimental_annoy_index>1</allow_experimental_annoy_index>
```

Or set via query:

```sql
SET allow_experimental_annoy_index = 1;
```

## Create a Table with annoy Index

```sql
CREATE TABLE embeddings (
    id UInt64,
    doc_id String,
    embedding Array(Float32),
    content String,
    INDEX idx_ann embedding TYPE annoy(100) GRANULARITY 1
) ENGINE = MergeTree()
ORDER BY id
SETTINGS index_granularity = 8192,
         allow_experimental_annoy_index = 1;
```

The parameter `100` is the number of trees - more trees means better accuracy but larger index size.

## Insert Embedding Data

```python
import clickhouse_connect
import numpy as np

client = clickhouse_connect.get_client(host="localhost")

rows = []
for i in range(10000):
    vec = np.random.rand(128).astype(np.float32).tolist()
    rows.append((i, f"doc-{i}", vec, f"Document content {i}"))

client.insert("embeddings", rows, column_names=["id", "doc_id", "embedding", "content"])
```

## Query Nearest Neighbors

```sql
WITH [0.1, 0.2, 0.3, ...] AS query_vec  -- 128-dim float array
SELECT
    id,
    doc_id,
    content,
    L2Distance(embedding, query_vec) AS distance
FROM embeddings
ORDER BY distance ASC
LIMIT 10;
```

## Full Example Query

```sql
SET allow_experimental_annoy_index = 1;

SELECT
    id,
    doc_id,
    L2Distance(embedding, [0.1, 0.2, 0.3, 0.4]) AS dist
FROM embeddings
ORDER BY dist ASC
LIMIT 5;
```

## Distance Functions

```sql
-- Euclidean (L2) distance
L2Distance(a, b)

-- Cosine distance
cosineDistance(a, b)

-- L1 (Manhattan) distance
L1Distance(a, b)
```

## Tune Number of Trees

```text
trees = 10   - fast index build, lower accuracy
trees = 50   - balanced (good default)
trees = 200  - high accuracy, slower build, larger index
```

```sql
INDEX idx_ann embedding TYPE annoy(50) GRANULARITY 1
```

## Check Index Disk Usage

```sql
SELECT
    formatReadableSize(sum(secondary_indices_bytes_on_disk)) AS ann_index_size
FROM system.parts
WHERE table = 'embeddings' AND active = 1;
```

## Limitations

```text
- annoy index is approximate (may miss some nearest neighbors)
- Only works with fixed-length Array(Float32) or Array(Float64)
- Requires allow_experimental_annoy_index = 1
- Not suitable for frequently updated vectors (index is static per part)
```

## Summary

The annoy skip index in ClickHouse enables approximate nearest neighbor search on vector embedding columns. It is well-suited for recommendation systems, semantic search, and similarity scoring at scale. Use 50-100 trees for a good balance of accuracy and index size, and prefer `cosineDistance` for normalized embeddings.
