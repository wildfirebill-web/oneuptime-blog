# How to Use $search in MongoDB Atlas Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Aggregation, Full-Text Search

Description: Learn how to use the $search stage in MongoDB Atlas aggregation pipelines to run full-text, fuzzy, phrase, and compound queries using the Lucene-based search engine.

---

## What Is the $search Stage?

The `$search` stage runs a full-text search query using MongoDB Atlas Search, which is powered by Apache Lucene. Unlike the standard `$text` operator, `$search` supports advanced features like fuzzy matching, autocomplete, phrase queries, compound logic, faceting, and relevance scoring. It must be the first stage in a pipeline and requires an Atlas Search index.

## Prerequisites

1. Your collection must be hosted on MongoDB Atlas
2. Create an Atlas Search index in the Atlas UI or via the Atlas CLI

```javascript
// Create a basic search index via Atlas CLI or Atlas UI
// Index definition:
{
  "mappings": {
    "dynamic": true
  }
}
```

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $search: {
      index: "<indexName>",
      <operator>: { ... }
    }
  }
])
```

## Example: Basic Text Search

```javascript
db.articles.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "mongodb aggregation tutorial",
        path: "content"
      }
    }
  },
  { $limit: 10 },
  { $project: { title: 1, score: { $meta: "searchScore" } } }
])
```

## Fuzzy Matching (Typo Tolerance)

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "wireles headphons",  // typos
        path: "name",
        fuzzy: { maxEdits: 1 }
      }
    }
  }
])
// Still finds "wireless headphones" despite typos
```

## Phrase Search

```javascript
db.articles.aggregate([
  {
    $search: {
      phrase: {
        query: "machine learning",
        path: "content"
      }
    }
  }
])
// Matches "machine learning" as an exact phrase, not individual words
```

## Compound Queries (AND, OR, NOT)

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          { text: { query: "headphones", path: "category" } }
        ],
        should: [
          { text: { query: "wireless", path: "description" } },
          { range: { path: "rating", gte: 4.0 } }
        ],
        mustNot: [
          { text: { query: "refurbished", path: "condition" } }
        ]
      }
    }
  }
])
```

## Range Queries

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        filter: [
          { range: { path: "price", gte: 50, lte: 200 } },
          { range: { path: "rating", gte: 4.0 } }
        ],
        must: [
          { text: { query: "bluetooth speaker", path: ["name", "description"] } }
        ]
      }
    }
  }
])
```

## Accessing Relevance Score

```javascript
db.articles.aggregate([
  {
    $search: {
      text: { query: "kubernetes monitoring", path: "content" }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $sort: { score: -1 } }
])
```

## Wildcard and Regex Queries

```javascript
db.logs.aggregate([
  {
    $search: {
      wildcard: {
        query: "error-*",
        path: "message",
        allowAnalyzedField: true
      }
    }
  }
])
```

## $search with Subsequent Pipeline Stages

```javascript
db.reviews.aggregate([
  {
    $search: {
      text: { query: "excellent battery life", path: "text" }
    }
  },
  { $match: { verified: true } },
  { $group: { _id: "$productId", avgRating: { $avg: "$rating" } } },
  { $sort: { avgRating: -1 } },
  { $limit: 10 }
])
```

## Summary

The `$search` stage brings Lucene-powered full-text search into MongoDB aggregation pipelines. It supports text, phrase, fuzzy, range, wildcard, and compound queries with relevance scoring. Pair it with subsequent aggregation stages for search-driven analytics. All Atlas Search functionality requires an Atlas deployment with a search index configured.
