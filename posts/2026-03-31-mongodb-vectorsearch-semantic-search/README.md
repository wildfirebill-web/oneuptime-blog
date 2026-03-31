# How to Use $vectorSearch for Semantic Search in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Vector Search, Semantic Search, Atlas, Embedding

Description: Use the $vectorSearch aggregation stage in MongoDB Atlas to perform semantic similarity search over vector embeddings for AI-powered applications.

---

## What Is Semantic Search?

Semantic search finds results based on meaning rather than exact keyword matches. Instead of searching for documents containing the word "automobile", semantic search also returns documents about "car", "vehicle", or "transportation" because their embeddings are close in vector space.

MongoDB Atlas enables semantic search through the `$vectorSearch` aggregation stage, which performs approximate nearest neighbor (ANN) search over stored embeddings.

## Basic $vectorSearch Query

```javascript
// Convert a user query to an embedding first
const queryEmbedding = await getEmbedding("best noise cancelling headphones");

const results = await db.collection("products").aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryEmbedding,
      numCandidates: 150,
      limit: 10
    }
  },
  {
    $project: {
      title: 1,
      description: 1,
      price: 1,
      score: { $meta: "vectorSearchScore" }
    }
  }
]).toArray();
```

## Getting Embeddings from OpenAI

```javascript
const { OpenAI } = require("openai");
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function getEmbedding(text) {
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: text
  });
  return response.data[0].embedding;
}
```

## Full Semantic Search Pipeline

```javascript
async function semanticSearch(db, query, options = {}) {
  const { category, limit = 10, minScore = 0.7 } = options;

  const queryVector = await getEmbedding(query);

  const pipeline = [
    {
      $vectorSearch: {
        index: "vector_index",
        path: "embedding",
        queryVector: queryVector,
        numCandidates: limit * 10,
        limit: limit,
        ...(category && {
          filter: { category: { $eq: category } }
        })
      }
    },
    {
      $project: {
        title: 1,
        description: 1,
        category: 1,
        score: { $meta: "vectorSearchScore" }
      }
    },
    {
      $match: {
        score: { $gte: minScore }
      }
    }
  ];

  return db.collection("articles").aggregate(pipeline).toArray();
}
```

## Hybrid Search: Combining Vector and Text Search

Combine semantic and keyword search with `$unionWith`:

```javascript
db.articles.aggregate([
  // Vector search results
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryEmbedding,
      numCandidates: 100,
      limit: 20
    }
  },
  {
    $group: {
      _id: null,
      docs: { $push: "$$ROOT" }
    }
  },
  {
    $unwind: {
      path: "$docs",
      includeArrayIndex: "rank"
    }
  },
  {
    $addFields: {
      "docs.vector_rank": { $add: ["$rank", 1] }
    }
  },
  {
    $replaceRoot: { newRoot: "$docs" }
  }
])
```

## Filtering with Pre-Filters

Use pre-filters to restrict the search space before vector comparison:

```javascript
db.products.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryVector,
      numCandidates: 200,
      limit: 10,
      filter: {
        $and: [
          { inStock: { $eq: true } },
          { price: { $lte: 500 } },
          { category: { $in: ["electronics", "audio"] } }
        ]
      }
    }
  },
  {
    $project: {
      title: 1,
      price: 1,
      score: { $meta: "vectorSearchScore" }
    }
  }
])
```

Note: Fields used in `filter` must be indexed as `filter` type in the vector index definition.

## Understanding numCandidates

`numCandidates` controls the trade-off between speed and accuracy:
- Higher values: better recall (more accurate), slower
- Lower values: faster, may miss some relevant results
- Recommended: set to 10-20x the `limit` value

```javascript
// For limit: 10, use numCandidates: 100-200
{
  $vectorSearch: {
    numCandidates: 150,
    limit: 10
  }
}
```

## Interpreting Vector Search Scores

Scores from `$vectorSearchScore` range from 0 to 1 for cosine similarity:
- 0.95+ : Very high similarity
- 0.85-0.95: High similarity
- 0.70-0.85: Moderate similarity
- Below 0.70: Likely not relevant

## Summary

`$vectorSearch` enables semantic search by finding documents whose embeddings are nearest to a query embedding. Generate query embeddings with an embedding model, set `numCandidates` to 10-20x your `limit`, and use pre-filters to narrow the search space. Combine with `$vectorSearchScore` thresholds to filter out low-relevance results from your response.
