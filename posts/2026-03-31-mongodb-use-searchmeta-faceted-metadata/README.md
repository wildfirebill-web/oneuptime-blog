# How to Use $searchMeta for Faceted Search Metadata in MongoDB Atlas

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Aggregation, Faceted Search

Description: Learn how to use MongoDB Atlas's $searchMeta stage to retrieve facet counts and search metadata without returning full documents, ideal for building search filter UIs.

---

## What Is the $searchMeta Stage?

The `$searchMeta` stage returns only the metadata produced by an Atlas Search query without returning the matching documents themselves. Its primary use case is retrieving facet counts for search filter sidebars - you get the category breakdowns and counts without the overhead of fetching all the matching documents. This makes it highly efficient for powering the filter panels in e-commerce and content search UIs.

## Prerequisites

- MongoDB Atlas cluster with Atlas Search enabled
- An Atlas Search index with facet mappings defined

```javascript
// Atlas Search index definition with facet fields
{
  "mappings": {
    "fields": {
      "category": [
        { "type": "stringFacet" },
        { "type": "string" }
      ],
      "price": [
        { "type": "numberFacet" },
        { "type": "number" }
      ],
      "brand": [{ "type": "stringFacet" }]
    }
  }
}
```

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $searchMeta: {
      index: "<indexName>",
      facet: {
        operator: { <searchOperator> },
        facets: {
          <facetName>: {
            type: "string" | "number" | "date",
            path: "<fieldPath>",
            numBuckets: <n>  // for number and date facets
          }
        }
      }
    }
  }
])
```

## Example: Facet Counts for an E-commerce Search

```javascript
db.products.aggregate([
  {
    $searchMeta: {
      facet: {
        operator: {
          text: { query: "headphones", path: "name" }
        },
        facets: {
          categoryFacet: {
            type: "string",
            path: "category"
          },
          brandFacet: {
            type: "string",
            path: "brand",
            numBuckets: 10
          },
          priceFacet: {
            type: "number",
            path: "price",
            boundaries: [0, 50, 100, 200, 500],
            default: "Other"
          }
        }
      }
    }
  }
])
```

Example output:
```javascript
{
  count: { lowerBound: 1247 },
  facets: {
    categoryFacet: {
      buckets: [
        { _id: "Over-Ear", count: 432 },
        { _id: "In-Ear", count: 389 },
        { _id: "On-Ear", count: 234 }
      ]
    },
    brandFacet: {
      buckets: [
        { _id: "Sony", count: 187 },
        { _id: "Bose", count: 145 }
      ]
    },
    priceFacet: {
      buckets: [
        { _id: 0, count: 89 },
        { _id: 50, count: 342 },
        { _id: 100, count: 456 },
        { _id: 200, count: 287 }
      ]
    }
  }
}
```

## Date Facets

```javascript
db.articles.aggregate([
  {
    $searchMeta: {
      facet: {
        operator: {
          text: { query: "kubernetes", path: "content" }
        },
        facets: {
          publishedFacet: {
            type: "date",
            path: "publishedAt",
            boundaries: [
              new Date("2023-01-01"),
              new Date("2024-01-01"),
              new Date("2025-01-01")
            ],
            default: "Other"
          }
        }
      }
    }
  }
])
```

## $searchMeta vs $search with $facet

```javascript
// Option 1: $searchMeta - metadata only, no documents
db.products.aggregate([
  { $searchMeta: { facet: { ... } } }
])

// Option 2: $search + $facet - get both documents and metadata
db.products.aggregate([
  { $search: { ... } },
  {
    $facet: {
      results: [{ $limit: 20 }],
      meta: [{ $replaceWith: "$$SEARCH_META" }, { $limit: 1 }]
    }
  }
])
```

`$searchMeta` is simpler when you only need the filter counts. Use `$$SEARCH_META` with `$replaceWith` when you need both results and counts in one pipeline.

## Accessing $$SEARCH_META Variable

After a `$search` stage, the variable `$$SEARCH_META` contains the same metadata that `$searchMeta` would return.

```javascript
db.products.aggregate([
  { $search: { text: { query: "shoes", path: "name" } } },
  { $limit: 20 },
  {
    $group: {
      _id: null,
      results: { $push: "$$ROOT" },
      meta: { $first: "$$SEARCH_META" }
    }
  }
])
```

## Summary

The `$searchMeta` stage retrieves Atlas Search facet counts and total result counts without returning document data. It is the most efficient way to power search filter sidebars, showing users how many results exist per category, brand, price range, or date range. For combined results and metadata, use `$$SEARCH_META` within a `$search`-based pipeline.
