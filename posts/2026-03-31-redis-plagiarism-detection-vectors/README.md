# How to Build a Plagiarism Detection System with Redis Vectors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector Search, Plagiarism, Similarity

Description: Detect plagiarism and near-duplicate text submissions using Redis Vector Search and sentence embeddings to compute semantic similarity scores.

---

Plagiarism detection needs to go beyond keyword matching to catch paraphrased or slightly modified content. Redis Vector Search enables semantic similarity comparison - if two documents mean the same thing, their embeddings will be close together regardless of exact wording.

## Installation

```bash
pip install redis sentence-transformers numpy
```

## Building the Document Corpus Index

```python
import redis
import numpy as np
from sentence_transformers import SentenceTransformer

r = redis.Redis(host='localhost', port=6379, decode_responses=False)
embedder = SentenceTransformer('all-MiniLM-L6-v2')

r.execute_command(
    'FT.CREATE', 'corpus_idx', 'ON', 'HASH',
    'PREFIX', '1', 'doc:',
    'SCHEMA',
    'title', 'TEXT',
    'author', 'TAG',
    'course', 'TAG',
    'embedding', 'VECTOR', 'HNSW', '6',
    'TYPE', 'FLOAT32', 'DIM', '384',
    'DISTANCE_METRIC', 'COSINE'
)
```

## Chunking Strategy for Long Documents

Split documents into overlapping chunks for better accuracy:

```python
def chunk_document(text: str, chunk_size: int = 200,
                   overlap: int = 50) -> list:
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        if len(chunk.split()) >= 30:  # skip very short chunks
            chunks.append(chunk)
    return chunks

def embed_document(text: str) -> bytes:
    # Embed the full document summary (first 512 words)
    summary = " ".join(text.split()[:512])
    vec = embedder.encode(summary, normalize_embeddings=True)
    return vec.astype(np.float32).tobytes()

def add_to_corpus(doc_id: str, title: str, text: str,
                  author: str, course: str):
    r.hset(f"doc:{doc_id}", mapping={
        "title": title.encode(),
        "author": author.encode(),
        "course": course.encode(),
        "embedding": embed_document(text)
    })
```

## Computing Document-Level Similarity

```python
def check_plagiarism(submission_text: str, top_k: int = 5,
                     course: str = None) -> list:
    query_vec = embed_document(submission_text)

    if course:
        query = (
            f"(@course:{{{course}}})"
            f"=>[KNN {top_k} @embedding $vec AS score]"
        )
    else:
        query = f"*=>[KNN {top_k} @embedding $vec AS score]"

    result = r.execute_command(
        'FT.SEARCH', 'corpus_idx', query,
        'PARAMS', 2, 'vec', query_vec,
        'RETURN', 4, 'title', 'author', 'course', 'score',
        'SORTBY', 'score',
        'DIALECT', 2
    )

    matches = []
    for i in range(1, len(result), 2):
        raw = result[i+1]
        fields = {}
        for j in range(0, len(raw), 2):
            fields[raw[j].decode()] = raw[j+1].decode()
        similarity = round(1 - float(fields.get('score', 1.0)), 4)
        matches.append({
            "title": fields.get('title', ''),
            "author": fields.get('author', ''),
            "course": fields.get('course', ''),
            "similarity_score": similarity,
            "is_suspected": similarity > 0.85
        })
    return matches
```

## Chunk-Level Analysis for Partial Plagiarism

Detect plagiarism in specific passages:

```python
def check_chunk_plagiarism(submission_text: str,
                            threshold: float = 0.90) -> list:
    chunks = chunk_document(submission_text)
    flagged = []

    for i, chunk in enumerate(chunks):
        query_vec = embedder.encode(chunk, normalize_embeddings=True)
        query_bytes = query_vec.astype(np.float32).tobytes()

        result = r.execute_command(
            'FT.SEARCH', 'corpus_idx',
            '*=>[KNN 3 @embedding $vec AS score]',
            'PARAMS', 2, 'vec', query_bytes,
            'RETURN', 3, 'title', 'author', 'score',
            'SORTBY', 'score',
            'DIALECT', 2
        )

        if result[0] > 0:
            raw = result[2]
            fields = {raw[j].decode(): raw[j+1].decode()
                      for j in range(0, len(raw), 2)}
            sim = 1 - float(fields.get('score', 1.0))
            if sim > threshold:
                flagged.append({
                    "chunk_index": i,
                    "chunk_preview": chunk[:100] + "...",
                    "matched_doc": fields.get('title', ''),
                    "similarity": round(sim, 4)
                })

    return flagged
```

## Generating a Plagiarism Report

```python
def generate_report(submission_text: str, submission_title: str,
                    course: str = None) -> dict:
    doc_matches = check_plagiarism(submission_text, course=course)
    chunk_flags = check_chunk_plagiarism(submission_text)

    top_match = doc_matches[0] if doc_matches else None
    overall_score = top_match['similarity_score'] if top_match else 0

    return {
        "submission": submission_title,
        "overall_similarity": overall_score,
        "verdict": "FLAGGED" if overall_score > 0.85 else "CLEAR",
        "top_matches": doc_matches[:3],
        "flagged_passages": len(chunk_flags),
        "passage_details": chunk_flags
    }
```

## Summary

Redis Vector Search enables semantic plagiarism detection that catches paraphrased content traditional tools miss. By embedding documents and chunks with sentence transformers and indexing them in Redis HNSW, you get sub-millisecond similarity checks. A two-level approach - document-level screening followed by chunk-level analysis - gives you both efficiency and precision for identifying the exact plagiarized passages.
