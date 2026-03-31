# How to Build RAG (Retrieval Augmented Generation) with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, RAG, Retrieval Augmented Generation, LLM, Vector Search, AI

Description: Learn how to build a Retrieval Augmented Generation (RAG) pipeline using ClickHouse as the vector store for semantic document retrieval.

---

Retrieval Augmented Generation (RAG) improves LLM responses by retrieving relevant documents from a knowledge base and including them as context in the prompt. ClickHouse makes an excellent vector store for RAG pipelines because it combines fast vector similarity search with powerful SQL filtering.

## RAG Architecture Overview

A RAG pipeline has two phases:

1. **Indexing** - Chunk documents, generate embeddings, and store them in ClickHouse.
2. **Querying** - Embed the user question, retrieve the most relevant chunks from ClickHouse, and pass them to the LLM.

## Setting Up the Knowledge Base Table

```sql
CREATE TABLE rag_knowledge_base
(
    chunk_id    UInt64,
    doc_id      UInt64,
    doc_title   String,
    chunk_text  String,
    chunk_index UInt16,
    embedding   Array(Float32),
    source_url  String,
    created_at  DateTime DEFAULT now(),
    INDEX emb_ann embedding TYPE usearch('cosineDistance') GRANULARITY 100000000
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (doc_id, chunk_index);
```

## Ingesting Documents

```python
import clickhouse_connect
from openai import OpenAI

ch = clickhouse_connect.get_client(host='localhost')
openai_client = OpenAI()

def chunk_text(text, size=500):
    words = text.split()
    return [' '.join(words[i:i+size]) for i in range(0, len(words), size)]

def embed(texts):
    resp = openai_client.embeddings.create(
        model='text-embedding-3-small',
        input=texts
    )
    return [r.embedding for r in resp.data]

doc = {'id': 1, 'title': 'ClickHouse Docs', 'url': 'https://clickhouse.com/docs'}
chunks = chunk_text("ClickHouse is a column-oriented database...")
embeddings = embed(chunks)

rows = [
    (i, doc['id'], doc['title'], chunk, i, emb, doc['url'])
    for i, (chunk, emb) in enumerate(zip(chunks, embeddings))
]
ch.insert('rag_knowledge_base',
          rows,
          column_names=['chunk_id','doc_id','doc_title','chunk_text',
                        'chunk_index','embedding','source_url'])
```

## Retrieving Relevant Chunks

Given a user question, embed it and retrieve top-k chunks:

```sql
-- Replace the array with the actual query embedding
SELECT
    chunk_text,
    doc_title,
    source_url,
    cosineDistance(embedding, [0.12, 0.84, 0.33, 0.57]) AS distance
FROM rag_knowledge_base
WHERE cosineDistance(embedding, [0.12, 0.84, 0.33, 0.57]) < 0.4
ORDER BY distance ASC
LIMIT 5;
```

## Combining Retrieval with Metadata Filters

Filter by source or recency to improve retrieval quality:

```sql
SELECT
    chunk_text,
    doc_title,
    cosineDistance(embedding, [0.12, 0.84, 0.33, 0.57]) AS distance
FROM rag_knowledge_base
WHERE source_url LIKE '%clickhouse.com%'
  AND created_at >= '2024-01-01'
  AND cosineDistance(embedding, [0.12, 0.84, 0.33, 0.57]) < 0.5
ORDER BY distance ASC
LIMIT 5;
```

## Passing Context to the LLM

```python
import clickhouse_connect
from openai import OpenAI

def rag_query(user_question: str, top_k: int = 5) -> str:
    openai_client = OpenAI()
    ch = clickhouse_connect.get_client(host='localhost')

    q_emb = openai_client.embeddings.create(
        model='text-embedding-3-small',
        input=[user_question]
    ).data[0].embedding

    emb_str = '[' + ','.join(str(x) for x in q_emb) + ']'
    rows = ch.query(f"""
        SELECT chunk_text, doc_title
        FROM rag_knowledge_base
        WHERE cosineDistance(embedding, {emb_str}) < 0.5
        ORDER BY cosineDistance(embedding, {emb_str}) ASC
        LIMIT {top_k}
    """).result_rows

    context = '\n\n'.join(f"[{r[1]}]: {r[0]}" for r in rows)
    prompt = f"Context:\n{context}\n\nQuestion: {user_question}\nAnswer:"

    return openai_client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[{'role': 'user', 'content': prompt}]
    ).choices[0].message.content
```

## Summary

ClickHouse is a capable RAG vector store that combines ANN index-accelerated similarity search with SQL-based metadata filtering. Store document chunks as `Array(Float32)` embeddings, index them with `usearch`, and retrieve the top-k chunks at query time to inject as LLM context. The hybrid filtering capability is a key advantage over dedicated vector databases.
