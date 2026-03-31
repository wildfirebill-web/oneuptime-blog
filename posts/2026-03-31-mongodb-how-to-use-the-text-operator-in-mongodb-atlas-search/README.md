# How to Use the text Operator in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Full-Text Search, Lucene

Description: Learn how to use the text operator in MongoDB Atlas Search to perform full-text search with analyzers, fuzzy matching, synonyms, and relevance scoring.

---

## Overview

MongoDB Atlas Search provides full-text search capabilities built on Apache Lucene. The `text` operator is the primary operator for searching text fields - it analyzes the query using a specified text analyzer, tokenizes it, and finds documents containing matching terms. Unlike regex or the standard MongoDB `$text` operator, Atlas Search text queries support relevance scoring, synonyms, fuzzy matching, and multi-language analyzers.

## Prerequisites

Atlas Search requires a MongoDB Atlas cluster. Create an Atlas Search index before running queries:

1. Go to your Atlas cluster
2. Navigate to Search Indexes
3. Click "Create Search Index"
4. Use the default configuration for a basic text index:

```json
{
  "mappings": {
    "dynamic": true
  }
}
```

Or create a specific mapping:

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": {
        "type": "string",
        "analyzer": "lucene.english"
      },
      "description": {
        "type": "string",
        "analyzer": "lucene.english"
      }
    }
  }
}
```

## Basic text Search

```javascript
// Search for documents containing "laptop"
db.products.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "laptop",
        path: "description"
      }
    }
  },
  {
    $limit: 10
  },
  {
    $project: {
      title: 1,
      description: 1,
      score: { $meta: "searchScore" }
    }
  }
])
```

The `$meta: "searchScore"` projection includes the relevance score for each result, allowing you to sort by relevance.

## Searching Multiple Fields

```javascript
// Search across multiple fields simultaneously
db.products.aggregate([
  {
    $search: {
      text: {
        query: "wireless bluetooth headphones",
        path: ["title", "description", "tags"]
      }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" }
    }
  },
  {
    $sort: { score: { $meta: "searchScore" } }
  }
])
```

## Phrase Search

Search for an exact phrase rather than individual tokens:

```javascript
db.products.aggregate([
  {
    $search: {
      phrase: {
        query: "noise cancelling headphones",
        path: "description"
      }
    }
  }
])
```

## Fuzzy Matching

Allow for typos and spelling variations:

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "wireles headphons",  // intentional typos
        path: "description",
        fuzzy: {
          maxEdits: 2,          // max character edits (1 or 2)
          prefixLength: 3,      // number of chars that must match exactly
          maxExpansions: 50     // max number of variations to consider
        }
      }
    }
  }
])
```

## Using Synonyms

First, create a synonym mapping collection and configure the search index to use it:

```javascript
// Create synonyms collection
db.synonyms.insertMany([
  {
    mappingType: "equivalent",
    synonyms: ["phone", "smartphone", "mobile", "handset"]
  },
  {
    mappingType: "equivalent",
    synonyms: ["laptop", "notebook", "computer"]
  }
])
```

Then configure the search index with the synonym source and run queries:

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "phone",
        path: "title",
        synonyms: "my_synonyms"  // name of the synonym mapping defined in index
      }
    }
  }
])
```

This would match documents containing "phone", "smartphone", "mobile", or "handset".

## Sorting by Relevance Score

Atlas Search results are already returned in relevance order by default. To explicitly sort:

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "gaming laptop",
        path: ["title", "description"]
      }
    }
  },
  {
    $addFields: {
      score: { $meta: "searchScore" }
    }
  },
  {
    $sort: { score: -1 }
  },
  {
    $limit: 20
  },
  {
    $project: { title: 1, score: 1 }
  }
])
```

## Combining text with Filters

Combine Atlas Search with regular MongoDB filters:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "laptop",
              path: "title"
            }
          }
        ],
        filter: [
          {
            range: {
              path: "price",
              gte: 500,
              lte: 2000
            }
          },
          {
            equals: {
              path: "inStock",
              value: true
            }
          }
        ]
      }
    }
  },
  {
    $limit: 10
  }
])
```

## Highlighting Search Terms

Return the matching text with hit highlights:

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "wireless headphones",
        path: "description"
      },
      highlight: {
        path: "description"
      }
    }
  },
  {
    $project: {
      title: 1,
      highlights: { $meta: "searchHighlights" }
    }
  }
])
```

Returns highlights like:

```javascript
{
  title: "Sony WH-1000XM5",
  highlights: [{
    path: "description",
    texts: [
      { value: "Industry-leading ", type: "text" },
      { value: "wireless", type: "hit" },
      { value: " noise-cancelling ", type: "text" },
      { value: "headphones", type: "hit" }
    ],
    score: 3.2
  }]
}
```

## Summary

The `text` operator in MongoDB Atlas Search provides powerful full-text search built on Lucene, with support for multi-field search, relevance scoring, fuzzy matching, synonyms, and phrase search. Combining `text` queries with the `compound` operator allows you to mix full-text search with structured filters like price ranges and availability. Atlas Search indexes must be created before queries work, but once in place, they enable search experiences far beyond what MongoDB's built-in `$text` operator can provide.
