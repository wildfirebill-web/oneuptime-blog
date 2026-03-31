# How to Build RAG (Retrieval Augmented Generation) with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, RAG, Retrieval Augmented Generation, Embedding, Vector, LLM

Description: Build a Retrieval Augmented Generation pipeline using ClickHouse as the vector store for embedding retrieval and context assembly for LLMs.

---

Retrieval Augmented Generation (RAG) grounds LLM responses in your own data by retrieving relevant context documents before generating an answer. ClickHouse serves as an efficient vector store for the retrieval step, especially when you need to combine embedding search with structured metadata filters.

## Architecture Overview

```text
User query
  -> Embed query (embedding model)
  -> Search ClickHouse for similar chunks (cosineDistance)
  -> Assemble context from top-K results
  -> Send [context + query] to LLM
  -> Return answer
```

## Schema: Document Chunks Table

```sql
CREATE TABLE knowledge_chunks
(
    chunk_id    UInt64,
    doc_id      UInt64,
    source      LowCardinality(String),
    content     String,
    embedding   Array(Float32),
    created_at  DateTime DEFAULT now()
)
ENGINE = MergeTree()
ORDER BY (source, chunk_id);
```

## Indexing Documents

```python
import clickhouse_connect
from openai import OpenAI

client_oai = OpenAI()
client_ch  = clickhouse_connect.get_client(host='localhost')

def embed(text):
    return client_oai.embeddings.create(
        input=text, model='text-embedding-3-small'
    ).data[0].embedding

chunks = [
    (1, 1, 'docs', 'ClickHouse uses MergeTree engines...', embed('ClickHouse uses MergeTree engines...')),
]
client_ch.insert('knowledge_chunks', chunks,
                 column_names=['chunk_id','doc_id','source','content','embedding'])
```

## Retrieval Query

```sql
WITH [/* query embedding */] AS q
SELECT
    chunk_id,
    content,
    cosineDistance(embedding, q) AS score
FROM knowledge_chunks
WHERE source = 'docs'
ORDER BY score ASC
LIMIT 5;
```

## Filtering by Metadata

Combine vector retrieval with structured filters for precision:

```sql
WITH [/* query embedding */] AS q
SELECT content, cosineDistance(embedding, q) AS score
FROM knowledge_chunks
WHERE source = 'docs'
  AND doc_id IN (SELECT doc_id FROM documents WHERE category = 'database')
ORDER BY score ASC
LIMIT 5;
```

## Assembling the Prompt

```python
def retrieve_context(query_text, source='docs', k=5):
    q_emb = embed(query_text)
    rows = client_ch.query(
        "WITH {q:Array(Float32)} AS q "
        "SELECT content FROM knowledge_chunks "
        "WHERE source = {src:String} "
        "ORDER BY cosineDistance(embedding, q) ASC LIMIT {k:UInt8}",
        parameters={'q': q_emb, 'src': source, 'k': k}
    ).result_rows
    return '\n\n'.join(r[0] for r in rows)
```

## Adding an ANN Index for Scale

For millions of chunks, add an approximate index:

```sql
ALTER TABLE knowledge_chunks
  ADD INDEX ann_idx embedding
  TYPE usearch('cosineDistance')
  GRANULARITY 2;

ALTER TABLE knowledge_chunks MATERIALIZE INDEX ann_idx;
```

## Summary

ClickHouse makes a practical RAG vector store because it combines embedding retrieval via `cosineDistance` with rich metadata filtering in a single SQL query. Index chunks with `Array(Float32)`, use ANN indexes for scale, and apply source or category filters to narrow the retrieval corpus before computing distances.
