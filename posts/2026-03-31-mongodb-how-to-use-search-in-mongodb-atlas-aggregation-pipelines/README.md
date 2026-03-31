# How to Use $search in MongoDB Atlas Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, $search, Full-Text Search, Aggregation, NoSQL

Description: Learn how to use MongoDB Atlas's $search aggregation stage to perform full-text search with relevance scoring, fuzzy matching, and compound queries.

---

## What Is the $search Stage?

The `$search` stage is available in MongoDB Atlas and enables full-text search powered by Apache Lucene. It provides relevance-based scoring, fuzzy matching, autocomplete, facets, and more - far beyond what MongoDB's standard `$text` operator offers.

`$search` must be the first stage in an aggregation pipeline.

## Prerequisites

1. Use a MongoDB Atlas cluster (M0 free tier supported).
2. Create an Atlas Search index on your collection.

```javascript
// Atlas Search index definition (JSON)
{
  "mappings": {
    "dynamic": true
  }
}
```

Or define specific field mappings:

```javascript
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": { "type": "string" },
      "description": { "type": "string" },
      "category": { "type": "stringFacet" },
      "price": { "type": "number" }
    }
  }
}
```

## Basic Text Search

Search articles by keyword:

```javascript
db.articles.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "mongodb aggregation pipeline",
        path: "title"
      }
    }
  },
  { $limit: 10 }
])
```

## Searching Multiple Fields

```javascript
db.products.aggregate([
  {
    $search: {
      text: {
        query: "wireless headphones",
        path: ["name", "description", "tags"]
      }
    }
  }
])
```

## Fuzzy Matching

Find documents even when the query has typos:

```javascript
db.articles.aggregate([
  {
    $search: {
      text: {
        query: "mongodatabase",  // typo of "mongodb database"
        path: "title",
        fuzzy: {
          maxEdits: 1,         // allow 1 character edit
          prefixLength: 3      // first 3 chars must match exactly
        }
      }
    }
  }
])
```

## Compound Queries

Combine multiple search clauses:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "laptop",
              path: "name"
            }
          }
        ],
        should: [
          {
            text: {
              query: "gaming",
              path: "description",
              score: { boost: { value: 2 } }
            }
          }
        ],
        mustNot: [
          {
            text: {
              query: "refurbished",
              path: "condition"
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
          }
        ]
      }
    }
  }
])
```

## Phrase Search

Match an exact phrase:

```javascript
db.articles.aggregate([
  {
    $search: {
      phrase: {
        query: "machine learning",
        path: "title"
      }
    }
  }
])
```

## Adding Relevance Score

Include the search score in results:

```javascript
db.articles.aggregate([
  {
    $search: {
      text: { query: "mongodb", path: "title" }
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

## Autocomplete Search

Power a search-as-you-type feature:

```javascript
// Requires an autocomplete index field mapping
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
  { $project: { name: 1, _id: 0 } }
])
```

## Range Query

Search within a numeric range:

```javascript
db.products.aggregate([
  {
    $search: {
      range: {
        path: "price",
        gte: 100,
        lte: 500
      }
    }
  }
])
```

## Practical Use Case - E-commerce Search

```javascript
function searchProducts(query, minPrice, maxPrice, page, pageSize) {
  return db.products.aggregate([
    {
      $search: {
        compound: {
          must: [
            { text: { query: query, path: ["name", "description"], fuzzy: { maxEdits: 1 } } }
          ],
          filter: [
            { range: { path: "price", gte: minPrice, lte: maxPrice } },
            { equals: { path: "inStock", value: true } }
          ]
        }
      }
    },
    {
      $project: {
        name: 1, price: 1, rating: 1,
        score: { $meta: "searchScore" }
      }
    },
    { $sort: { score: -1 } },
    { $skip: (page - 1) * pageSize },
    { $limit: pageSize }
  ]);
}
```

## Summary

The `$search` stage in MongoDB Atlas brings enterprise-grade full-text search capabilities directly into the aggregation pipeline. With support for relevance scoring, fuzzy matching, compound boolean queries, autocomplete, and facets, it enables sophisticated search experiences that would otherwise require a separate search engine. Always define explicit field mappings for production use to control which fields are indexed.
