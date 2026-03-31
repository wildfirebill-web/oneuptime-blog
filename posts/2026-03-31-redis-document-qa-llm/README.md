# How to Build a Document Q&A System with Redis and LLMs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, LLM, Document, Q&A

Description: Build a document question-and-answer system using Redis Vector Search for retrieval and an LLM for answer generation with source citations.

---

A document Q&A system lets users ask natural language questions and receive answers grounded in specific documents. Redis handles the fast vector retrieval while an LLM synthesizes the answer from retrieved passages.

## Architecture

- Documents are chunked, embedded, and stored in Redis
- User questions are embedded and matched against stored chunks
- Retrieved chunks are passed to the LLM as context
- The LLM generates an answer and cites the source documents

## Installation

```bash
pip install redis sentence-transformers openai pypdf numpy
```

## Step 1 - Building the Index

```python
import redis
import numpy as np
from sentence_transformers import SentenceTransformer

r = redis.Redis(host='localhost', port=6379, decode_responses=False)
embedder = SentenceTransformer('all-MiniLM-L6-v2')

r.execute_command(
    'FT.CREATE', 'qa_idx', 'ON', 'HASH',
    'PREFIX', '1', 'qa_chunk:',
    'SCHEMA',
    'text', 'TEXT',
    'doc_name', 'TAG',
    'page', 'NUMERIC',
    'embedding', 'VECTOR', 'HNSW', '6',
    'TYPE', 'FLOAT32', 'DIM', '384',
    'DISTANCE_METRIC', 'COSINE'
)
```

## Step 2 - Ingesting PDF Documents

```python
import pypdf
import hashlib

def extract_pdf_chunks(pdf_path: str, chunk_size: int = 400):
    reader = pypdf.PdfReader(pdf_path)
    chunks = []
    for page_num, page in enumerate(reader.pages):
        text = page.extract_text() or ""
        words = text.split()
        for i in range(0, len(words), chunk_size - 50):
            chunk = " ".join(words[i:i + chunk_size])
            if len(chunk.strip()) > 50:
                chunks.append({
                    "text": chunk,
                    "page": page_num + 1
                })
    return chunks

def ingest_pdf(pdf_path: str, doc_name: str):
    chunks = extract_pdf_chunks(pdf_path)
    pipe = r.pipeline()
    for i, chunk in enumerate(chunks):
        vec = embedder.encode(chunk["text"], normalize_embeddings=True)
        chunk_id = hashlib.md5(f"{doc_name}:{i}".encode()).hexdigest()
        pipe.hset(f"qa_chunk:{chunk_id}", mapping={
            "text": chunk["text"].encode(),
            "doc_name": doc_name.encode(),
            "page": chunk["page"],
            "embedding": vec.astype(np.float32).tobytes()
        })
    pipe.execute()
    return len(chunks)
```

## Step 3 - Retrieving Relevant Passages

```python
def retrieve_passages(question: str, top_k: int = 4,
                      doc_name: str = None):
    query_vec = embedder.encode(question, normalize_embeddings=True)
    query_bytes = query_vec.astype(np.float32).tobytes()

    if doc_name:
        search_query = (
            f"(@doc_name:{{{doc_name}}})"
            f"=>[KNN {top_k} @embedding $vec AS score]"
        )
    else:
        search_query = f"*=>[KNN {top_k} @embedding $vec AS score]"

    result = r.execute_command(
        'FT.SEARCH', 'qa_idx', search_query,
        'PARAMS', 2, 'vec', query_bytes,
        'RETURN', 4, 'text', 'doc_name', 'page', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    passages = []
    for i in range(1, len(result), 2):
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        passages.append(fields)
    return passages
```

## Step 4 - Generating Answers with Citations

```python
import openai

client = openai.OpenAI()

def answer_question(question: str, doc_name: str = None) -> dict:
    passages = retrieve_passages(question, top_k=4, doc_name=doc_name)

    context_parts = []
    for p in passages:
        context_parts.append(
            f"[Source: {p['doc_name']}, Page {p['page']}]\n{p['text']}"
        )
    context = "\n\n---\n\n".join(context_parts)

    prompt = f"""Answer the question using only the provided context.
Include source citations (document name and page number).

Context:
{context}

Question: {question}

Answer:"""

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.1
    )

    return {
        "answer": response.choices[0].message.content,
        "sources": [{"doc": p['doc_name'], "page": p['page']}
                    for p in passages]
    }

result = answer_question("What are the main risk factors?",
                         doc_name="annual_report_2025")
print(result["answer"])
print("Sources:", result["sources"])
```

## Summary

Redis Vector Search combined with an LLM creates a powerful document Q&A system that retrieves precise passages and generates grounded answers with citations. Chunking documents, embedding with a sentence transformer, and using Redis HNSW search keeps retrieval latency under 10ms even for large document collections. Always pass source citations back to users so they can verify answers against the original text.
