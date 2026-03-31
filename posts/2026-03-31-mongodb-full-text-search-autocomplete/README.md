# How to Implement Full-Text Search with Autocomplete in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Search, Autocomplete, Atlas Search, Text Index

Description: Learn how to implement full-text search with autocomplete in MongoDB using Atlas Search autocomplete operators and text indexes for self-hosted deployments.

---

## Overview

Full-text search with autocomplete allows users to find results as they type. MongoDB provides two approaches: Atlas Search (cloud-only) with powerful autocomplete operators, and text indexes (available on all deployments) with manual prefix matching for self-hosted clusters.

## Option 1: Atlas Search Autocomplete

Create an Atlas Search index with autocomplete field mapping:

```json
{
  "mappings": {
    "fields": {
      "name": [
        {
          "type": "autocomplete",
          "analyzer": "lucene.standard",
          "tokenization": "edgeGram",
          "minGrams": 2,
          "maxGrams": 15
        },
        {
          "type": "string"
        }
      ],
      "description": {
        "type": "string",
        "analyzer": "lucene.standard"
      }
    }
  }
}
```

Query with the `autocomplete` operator:

```javascript
db.products.aggregate([
  { $search: {
      index: "products_search",
      compound: {
        must: [{
          autocomplete: {
            query: req.query.q,
            path: "name",
            fuzzy: { maxEdits: 1, prefixLength: 2 }
          }
        }]
      }
  }},
  { $limit: 10 },
  { $project: {
      name: 1,
      category: 1,
      price: 1,
      score: { $meta: "searchScore" }
  }}
])
```

## Combining Autocomplete with Full-Text Search

```javascript
db.products.aggregate([
  { $search: {
      index: "products_search",
      compound: {
        should: [
          {
            autocomplete: {
              query: searchTerm,
              path: "name",
              score: { boost: { value: 3 } }
            }
          },
          {
            text: {
              query: searchTerm,
              path: "description"
            }
          }
        ]
      }
  }},
  { $limit: 20 },
  { $sort: { score: { $meta: "searchScore" } } }
])
```

## Option 2: Text Indexes (Self-Hosted)

For self-hosted MongoDB, create a text index:

```javascript
db.products.createIndex({ name: "text", description: "text" }, {
  weights: { name: 10, description: 1 }
})
```

Full-text search with a text index:

```javascript
db.products.find(
  { $text: { $search: "wireless headphones" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } }).limit(10)
```

## Prefix Matching for Autocomplete Without Atlas

For prefix-based autocomplete without Atlas Search, use a regex index:

```javascript
// Create an index on the name field for prefix queries
db.products.createIndex({ name: 1 })

// Autocomplete with regex prefix match
db.products.find(
  { name: { $regex: `^${escapeRegex(prefix)}`, $options: "i" } }
).limit(10).project({ name: 1 })

function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}
```

## Implementing a Search API Endpoint

```javascript
app.get("/api/search", async (req, res) => {
  const { q, limit = 10 } = req.query;
  if (!q || q.trim().length < 2) return res.json([]);

  const results = await db.collection("products").aggregate([
    { $search: {
        autocomplete: { query: q, path: "name", fuzzy: { maxEdits: 1 } }
    }},
    { $limit: Number(limit) },
    { $project: { name: 1, category: 1, price: 1 } }
  ]).toArray();

  res.json(results);
});
```

## Summary

Implement full-text search with autocomplete in MongoDB using Atlas Search's `autocomplete` operator for the best relevance and fuzzy matching on cloud deployments. For self-hosted clusters, use text indexes for full-text search and prefix regex queries for autocomplete. Combine both operators in a `compound` query with score boosting to surface the most relevant results first.
