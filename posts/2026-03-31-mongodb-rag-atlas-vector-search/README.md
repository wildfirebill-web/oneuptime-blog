# How to Build a RAG Application with MongoDB Atlas Vector Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Vector Search, AI

Description: Build a Retrieval-Augmented Generation application using MongoDB Atlas Vector Search to retrieve relevant context and OpenAI to generate grounded responses.

---

Retrieval-Augmented Generation (RAG) combines a vector search retrieval step with a language model generation step. Instead of relying on the LLM's training data alone, you retrieve relevant documents from your own data store and inject them as context before generating an answer.

## Architecture

```text
User Query
   -> Embed query with OpenAI
   -> $vectorSearch in MongoDB
   -> Top-K relevant chunks returned
   -> Inject chunks into LLM prompt
   -> LLM generates grounded answer
```

## Setup

```bash
pip install pymongo openai
```

## Step 1: Prepare and Store Document Chunks

Chunk your documents into segments of 200-500 words for better retrieval precision:

```python
from openai import OpenAI
from pymongo import MongoClient

client = OpenAI(api_key="YOUR_KEY")
mongo = MongoClient("mongodb+srv://user:pass@cluster.mongodb.net/")
collection = mongo["rag_db"]["documents"]

def chunk_text(text: str, chunk_size: int = 400) -> list[str]:
    words = text.split()
    return [" ".join(words[i:i+chunk_size]) for i in range(0, len(words), chunk_size)]

def embed(text: str) -> list[float]:
    return client.embeddings.create(
        input=[text],
        model="text-embedding-3-small"
    ).data[0].embedding

# Index a document
doc_text = "MongoDB Atlas is a multi-cloud developer data platform..."
chunks = chunk_text(doc_text)

chunk_docs = [
    {
        "source": "atlas-overview",
        "chunk_index": i,
        "text": chunk,
        "embedding": embed(chunk)
    }
    for i, chunk in enumerate(chunks)
]
collection.insert_many(chunk_docs)
```

## Step 2: Create the Vector Search Index

```json
{
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1536,
      "similarity": "cosine"
    },
    {
      "type": "filter",
      "path": "source"
    }
  ]
}
```

## Step 3: Retrieve Relevant Chunks

```python
def retrieve_context(query: str, top_k: int = 4, source: str = None) -> list[dict]:
    query_vec = embed(query)

    pipeline = [
        {
            "$vectorSearch": {
                "index": "rag_index",
                "path": "embedding",
                "queryVector": query_vec,
                "numCandidates": 50,
                "limit": top_k,
                **({"filter": {"source": source}} if source else {})
            }
        },
        {
            "$project": {
                "text": 1,
                "source": 1,
                "score": {"$meta": "vectorSearchScore"}
            }
        }
    ]
    return list(collection.aggregate(pipeline))
```

## Step 4: Generate an Answer with Context

```python
def rag_answer(question: str) -> str:
    chunks = retrieve_context(question, top_k=4)
    context = "\n\n".join([c["text"] for c in chunks])

    prompt = f"""Answer the question using only the context below.
If the context does not contain the answer, say "I don't know."

Context:
{context}

Question: {question}
Answer:"""

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0
    )
    return response.choices[0].message.content

print(rag_answer("What is MongoDB Atlas and what databases does it support?"))
```

## Step 5: Add Source Attribution

Track which chunks contributed to the answer:

```python
def rag_answer_with_sources(question: str) -> dict:
    chunks = retrieve_context(question, top_k=4)
    context = "\n\n".join([c["text"] for c in chunks])
    sources = list({c["source"] for c in chunks})

    # ... generate answer as above ...

    return {
        "answer": answer,
        "sources": sources,
        "chunks_used": len(chunks)
    }
```

## Summary

A MongoDB RAG application stores document chunks with their embeddings in Atlas, uses `$vectorSearch` to retrieve the most relevant chunks for each user question, and injects those chunks as context into an LLM prompt. This pattern grounds the LLM response in your actual data, reduces hallucinations, and makes the knowledge base easy to update by just inserting new documents.
