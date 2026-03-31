# What Is MongoDB Atlas Search and How It Differs from Text Indexes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Text Index, Full-Text Search, Lucene

Description: MongoDB Atlas Search is a full-featured search engine powered by Apache Lucene that offers far richer query capabilities than MongoDB's built-in text indexes.

---

## Overview

MongoDB has had built-in text indexes since version 2.4 - these allow basic full-text search with stemming and stop words. Atlas Search, introduced on MongoDB Atlas, is an entirely different and far more capable system powered by Apache Lucene. It supports relevance scoring, autocomplete, fuzzy matching, faceted navigation, custom analyzers, and geospatial queries - features that are difficult or impossible with standard text indexes.

## MongoDB Text Indexes

Text indexes are created with `createIndex` and searched with the `$text` operator:

```javascript
// Create a text index
db.articles.createIndex({ title: "text", body: "text" })

// Query with $text
db.articles.find({ $text: { $search: "mongodb performance tuning" } })
```

Limitations of text indexes:
- Only equality matching with stemming - no fuzzy matching, no phrase proximity
- Case-insensitive but no language analyzer support beyond built-in languages
- No relevance scoring beyond the basic `textScore`
- No autocomplete support
- Only one text index per collection
- No faceted counts

## Atlas Search Overview

Atlas Search creates a separate Lucene index synced to your MongoDB collection. You define search indexes using JSON, and query them using the `$search` aggregation stage.

```javascript
// Query using Atlas Search
db.articles.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "mongodb performance",
        path: ["title", "body"],
        fuzzy: { maxEdits: 1 }
      }
    }
  },
  { $limit: 10 },
  { $project: { title: 1, score: { $meta: "searchScore" } } }
])
```

## Feature Comparison

| Feature | Text Index | Atlas Search |
|---|---|---|
| Basic full-text search | Yes | Yes |
| Fuzzy matching | No | Yes |
| Autocomplete | No | Yes (edge n-gram) |
| Phrase search | Limited | Yes |
| Relevance scoring | Basic | Rich (BM25 + boosting) |
| Custom analyzers | No | Yes |
| Faceted counts | No | Yes |
| Geospatial search | No | Yes |
| Multi-language | Limited | Extensive |
| Synonym support | No | Yes |
| Highlighting | No | Yes |
| Deployment | Any MongoDB | Atlas only |

## Creating an Atlas Search Index

Atlas Search indexes are created via the Atlas UI, Atlas CLI, or API - not with `createIndex`:

```bash
# Atlas CLI
atlas clusters search indexes create \
  --clusterName myCluster \
  --file search-index.json
```

```json
{
  "name": "default",
  "mappings": {
    "dynamic": true,
    "fields": {
      "title": {
        "type": "string",
        "analyzer": "lucene.standard"
      },
      "tags": {
        "type": "stringFacet"
      }
    }
  }
}
```

## Autocomplete with Atlas Search

```javascript
db.products.aggregate([
  {
    $search: {
      autocomplete: {
        query: "wire",
        path: "name",
        fuzzy: { maxEdits: 1 }
      }
    }
  },
  { $limit: 5 },
  { $project: { name: 1 } }
])
```

## When to Use Each

**Use text indexes when:**
- You need basic keyword search
- You are not on Atlas (self-hosted MongoDB)
- You have simple requirements and want minimal setup

**Use Atlas Search when:**
- You need fuzzy matching, autocomplete, or facets
- You are building a search-as-you-type experience
- You need relevance tuning and scoring boosts
- You want synonym and custom analyzer support

## Summary

MongoDB text indexes provide basic full-text search for simple use cases. Atlas Search, powered by Apache Lucene, delivers a full search engine experience with fuzzy matching, autocomplete, facets, custom analyzers, and rich relevance scoring. If you are on Atlas and building any significant search feature, Atlas Search is the right choice.
