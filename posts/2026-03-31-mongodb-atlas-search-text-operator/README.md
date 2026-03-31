# How to Use the text Operator in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Full-Text Search, Index, Query

Description: Use the Atlas Search text operator to perform full-text searches with analyzers, synonyms, and relevance scoring in MongoDB Atlas.

---

## What Is the Atlas Search text Operator?

The `text` operator in MongoDB Atlas Search performs full-text search on indexed string fields. Unlike MongoDB's native `$text` operator, Atlas Search's `text` operator supports advanced analyzers, multi-language search, synonym mappings, and relevance scoring through Lucene under the hood.

## Setting Up an Atlas Search Index

Before using the `text` operator, create an Atlas Search index in the Atlas UI or via the API:

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
db.products.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "wireless headphones",
        path: "title"
      }
    }
  },
  { $limit: 10 },
  { $project: { title: 1, description: 1, score: { $meta: "searchScore" } } }
])
```

## Search Across Multiple Fields

Use an array for `path` to search across multiple fields:

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "wireless headphones",
        path: ["title", "description", "tags"]
      }
    }
  },
  { $limit: 10 }
])
```

Or use a wildcard path:

```javascript
{
  $search: {
    text: {
      query: "wireless",
      path: { wildcard: "*" }
    }
  }
}
```

## Using Analyzers

Specify an analyzer to control tokenization and normalization:

```javascript
db.articles.aggregate([
  {
    $search: {
      text: {
        query: "running shoes",
        path: "content",
        synonyms: "product_synonyms"
      }
    }
  }
])
```

Available built-in analyzers include `lucene.standard`, `lucene.english`, `lucene.french`, `lucene.spanish`, and more.

## Fuzzy Search with text

Enable fuzzy matching to handle typos:

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "wireles headphnes",
        path: "title",
        fuzzy: {
          maxEdits: 2,
          prefixLength: 3
        }
      }
    }
  }
])
```

## Score Boosting

Boost relevance for matches in specific fields:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        should: [
          {
            text: {
              query: "wireless headphones",
              path: "title",
              score: { boost: { value: 3 } }
            }
          },
          {
            text: {
              query: "wireless headphones",
              path: "description",
              score: { boost: { value: 1 } }
            }
          }
        ]
      }
    }
  },
  { $project: { title: 1, score: { $meta: "searchScore" } } }
])
```

## Retrieving Search Scores

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "laptop",
        path: ["title", "category"]
      }
    }
  },
  {
    $project: {
      title: 1,
      category: 1,
      searchScore: { $meta: "searchScore" }
    }
  },
  { $sort: { searchScore: -1 } },
  { $limit: 5 }
])
```

## Summary

The Atlas Search `text` operator enables powerful full-text search on MongoDB collections with Lucene-based analyzers. Use multi-field `path` arrays for broad searches, `fuzzy` for typo tolerance, and score boosting in `compound` queries to prioritize title matches over body text. Always create a dedicated Atlas Search index with appropriate field mappings before querying.
