# How to Set Up Atlas Vector Search Indexes in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Vector Search, Index, AI

Description: Set up Atlas Vector Search indexes in MongoDB to enable semantic similarity search over embeddings stored in your collections.

---

## What Is Atlas Vector Search?

MongoDB Atlas Vector Search enables nearest-neighbor search over high-dimensional vector embeddings stored in your documents. This powers semantic search, recommendation systems, and RAG (retrieval-augmented generation) applications by finding documents similar in meaning rather than exact keyword matches.

## Prerequisites

- MongoDB Atlas M10 or higher cluster (or M0 for testing)
- MongoDB Atlas cluster running MongoDB 6.0.11+ or 7.0.2+
- Embeddings pre-computed and stored in your documents

## Storing Embeddings in Documents

Your documents should contain a field with the embedding array:

```javascript
db.articles.insertMany([
  {
    title: "Introduction to Machine Learning",
    content: "Machine learning is a subset of artificial intelligence...",
    embedding: [0.023, -0.045, 0.112, ...] // 1536-dimension vector
  }
])
```

## Creating a Vector Search Index

### Via the Atlas UI

1. Navigate to your cluster and click **Browse Collections**
2. Select your collection and click the **Search Indexes** tab
3. Click **Create Index** and choose **JSON Editor**
4. Paste the index definition and click **Create Search Index**

### Index Definition

```json
{
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1536,
      "similarity": "cosine"
    }
  ]
}
```

For additional filterable fields, include them as `filter` fields:

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
      "path": "category"
    },
    {
      "type": "filter",
      "path": "publishedAt"
    }
  ]
}
```

### Via the Atlas CLI

```bash
atlas clusters search indexes create \
  --clusterName myCluster \
  --db articles \
  --collection articles \
  --file vector-index.json
```

### Via the Atlas Administration API

```bash
curl -u "publicKey:privateKey" \
  --digest \
  -X POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/clusters/{clusterName}/fts/indexes" \
  -H "Content-Type: application/json" \
  -d '{
    "collectionName": "articles",
    "database": "myapp",
    "name": "vector_index",
    "type": "vectorSearch",
    "fields": [
      {
        "type": "vector",
        "path": "embedding",
        "numDimensions": 1536,
        "similarity": "cosine"
      }
    ]
  }'
```

## Similarity Metrics

| Metric | When to Use |
|---|---|
| `cosine` | Text embeddings - measures angle between vectors |
| `euclidean` | Numerical features where magnitude matters |
| `dotProduct` | When vectors are normalized to unit length |

Most embedding models (OpenAI, Cohere) work best with `cosine` similarity.

## Dimension Counts by Model

| Embedding Model | Dimensions |
|---|---|
| OpenAI text-embedding-3-small | 1536 |
| OpenAI text-embedding-3-large | 3072 |
| Cohere embed-english-v3.0 | 1024 |
| Sentence-transformers all-MiniLM-L6-v2 | 384 |

## Checking Index Status

```bash
atlas clusters search indexes list \
  --clusterName myCluster \
  --db myapp \
  --collection articles
```

Wait for the status to show `READY` before querying.

## Verifying with a Test Query

```javascript
db.articles.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: [0.023, -0.045, 0.112 /* ...1536 dims */],
      numCandidates: 100,
      limit: 5
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "vectorSearchScore" }
    }
  }
])
```

## Summary

Setting up Atlas Vector Search requires storing pre-computed embeddings in your documents, creating a `vectorSearch` index with the correct `numDimensions` and `similarity` metric, and waiting for the index to reach `READY` status. Choose `cosine` similarity for text embeddings from most LLM providers, and add `filter` fields to the index for hybrid filtering during search.
