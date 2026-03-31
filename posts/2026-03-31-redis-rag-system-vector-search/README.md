# How to Build a RAG System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RAG, Vector Search, LLM

Description: Build a Retrieval-Augmented Generation system using Redis Vector Search to ground LLM responses in your own documents.

---

Retrieval-Augmented Generation (RAG) improves LLM accuracy by retrieving relevant documents from your knowledge base and injecting them into the prompt. Redis Vector Search serves as the high-speed retrieval layer.

## Architecture

The RAG pipeline has three stages:
1. Ingest: chunk documents, embed, store in Redis
2. Retrieve: embed the user query and find similar chunks
3. Generate: pass chunks as context to the LLM

## Installation

```bash
pip install redis sentence-transformers openai numpy
```

## Step 1 - Ingesting and Chunking Documents

```python
import redis
import numpy as np
from sentence_transformers import SentenceTransformer

r = redis.Redis(host='localhost', port=6379, decode_responses=False)
embedder = SentenceTransformer('all-MiniLM-L6-v2')

# Create vector index
r.execute_command(
    'FT.CREATE', 'rag_idx', 'ON', 'HASH', 'PREFIX', '1', 'chunk:',
    'SCHEMA',
    'text', 'TEXT',
    'source', 'TAG',
    'chunk_id', 'NUMERIC',
    'embedding', 'VECTOR', 'HNSW', '6',
    'TYPE', 'FLOAT32', 'DIM', '384', 'DISTANCE_METRIC', 'COSINE'
)

def chunk_text(text: str, chunk_size: int = 300, overlap: int = 50):
    words = text.split()
    chunks = []
    start = 0
    while start < len(words):
        end = start + chunk_size
        chunks.append(" ".join(words[start:end]))
        start += chunk_size - overlap
    return chunks

def ingest_document(source: str, content: str):
    chunks = chunk_text(content)
    pipe = r.pipeline()
    for i, chunk in enumerate(chunks):
        vec = embedder.encode(chunk, normalize_embeddings=True)
        chunk_key = f"chunk:{source}:{i}"
        pipe.hset(chunk_key, mapping={
            "text": chunk.encode(),
            "source": source.encode(),
            "chunk_id": i,
            "embedding": vec.astype(np.float32).tobytes()
        })
    pipe.execute()
    return len(chunks)
```

## Step 2 - Retrieving Relevant Chunks

```python
def retrieve(query: str, top_k: int = 3):
    query_vec = embedder.encode(query, normalize_embeddings=True)
    query_bytes = query_vec.astype(np.float32).tobytes()

    result = r.execute_command(
        'FT.SEARCH', 'rag_idx',
        f'*=>[KNN {top_k} @embedding $vec AS score]',
        'PARAMS', 2, 'vec', query_bytes,
        'RETURN', 3, 'text', 'source', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    chunks = []
    for i in range(1, len(result), 2):
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        chunks.append(fields)
    return chunks
```

## Step 3 - Generating the Answer

```python
import openai

client = openai.OpenAI()

def rag_answer(question: str) -> str:
    chunks = retrieve(question, top_k=3)
    context = "\n\n".join(c['text'] for c in chunks)

    prompt = f"""Answer the question using only the context below.

Context:
{context}

Question: {question}
Answer:"""

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

answer = rag_answer("What is the recommended chunk size for RAG?")
print(answer)
```

## Caching Repeated Queries

Cache answers to avoid redundant LLM calls:

```python
import hashlib

def cached_rag_answer(question: str, ttl: int = 3600) -> str:
    cache_key = f"rag:cache:{hashlib.md5(question.encode()).hexdigest()}"
    cached = r.get(cache_key)
    if cached:
        return cached.decode()

    answer = rag_answer(question)
    r.setex(cache_key, ttl, answer)
    return answer
```

## Summary

Redis provides a unified platform for RAG: it stores vector embeddings for semantic retrieval, caches LLM responses to reduce cost, and handles all operations with sub-millisecond latency. By chunking your documents, embedding them with a sentence transformer, and using Redis Vector Search for retrieval, you can build a grounded question-answering system that stays accurate and cost-efficient.
