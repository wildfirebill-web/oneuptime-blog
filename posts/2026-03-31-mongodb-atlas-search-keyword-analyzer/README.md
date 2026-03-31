# How to Use the Keyword Analyzer in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Analyzer, Keyword, Exact Match

Description: Learn how the keyword analyzer works in MongoDB Atlas Search for exact-match search, faceting, and sorting by treating the entire field value as a single token.

---

## What Is the Keyword Analyzer?

The keyword analyzer in MongoDB Atlas Search (based on Lucene's KeywordAnalyzer) treats the entire field value as a single, unmodified token. It performs no tokenization, no lowercasing, and no filtering. The complete string is indexed as-is.

```text
Input: "New York, NY"
Keyword analyzer token: ["New York, NY"]  (one token, entire string)

Input: "mongodb-atlas"
Keyword analyzer token: ["mongodb-atlas"]  (no splitting on hyphens)
```

## When to Use the Keyword Analyzer

The keyword analyzer is the right choice for:

- **Exact match search** on identifiers, codes, and slugs
- **Faceted search** on category values, status fields, and enum fields
- **Sorting** by a text field (Atlas Search requires a keyword-analyzed field for sort)
- **Auto-suggest** on structured values like country codes or product SKUs
- Fields where the entire value is the searchable unit

## Configuring a Keyword Analyzer Index

```javascript
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "status": {
        "type": "string",
        "analyzer": "lucene.keyword"
      },
      "countryCode": {
        "type": "string",
        "analyzer": "lucene.keyword"
      },
      "sku": {
        "type": "string",
        "analyzer": "lucene.keyword"
      }
    }
  }
}
```

## Exact Match Queries

```javascript
// Find all orders with status "shipped"
db.orders.aggregate([
  {
    $search: {
      index: "orders_search",
      text: {
        query: "shipped",
        path: "status"
      }
    }
  },
  { $project: { orderId: 1, status: 1 } }
])
```

Because `"shipped"` is indexed as a single keyword token, this query matches exactly `"shipped"` and not `"shipping"` or `"ship"`.

## Faceted Search with the Keyword Analyzer

Facets require keyword-analyzed string fields to count distinct values:

```javascript
db.products.aggregate([
  {
    $searchMeta: {
      index: "products_search",
      facet: {
        operator: {
          text: { query: "laptop", path: "title" }
        },
        facets: {
          brandFacet:    { type: "string", path: "brand" },
          categoryFacet: { type: "string", path: "category" },
          statusFacet:   { type: "string", path: "status" }
        }
      }
    }
  }
])
```

The `brand`, `category`, and `status` fields must be indexed with `lucene.keyword` for faceting to work.

## Sorting with the Keyword Analyzer

Atlas Search uses a keyword-analyzed field for text sorting:

```javascript
{
  "mappings": {
    "fields": {
      "title": [
        {
          "type": "string",
          "analyzer": "lucene.standard"  // for search
        },
        {
          "type": "string",
          "analyzer": "lucene.keyword",
          "name": "title.keyword"        // for sorting
        }
      ]
    }
  }
}
```

Sort by the keyword sub-field:

```javascript
db.articles.aggregate([
  { $search: { index: "articles_search", text: { query: "mongodb", path: "title" } } },
  { $sort: { "title.keyword": 1 } }
])
```

## Combining Keyword with Standard Analyzer

A common pattern is to index the same field twice - once with standard for search, once with keyword for faceting and sorting:

```javascript
{
  "mappings": {
    "fields": {
      "category": [
        {
          "type": "string",
          "analyzer": "lucene.standard"
        },
        {
          "type": "string",
          "analyzer": "lucene.keyword",
          "name": "category.keyword"
        }
      ]
    }
  }
}
```

## Summary

The MongoDB Atlas Search keyword analyzer indexes the entire field value as a single unmodified token. It is the right choice for exact-match queries on identifiers and codes, faceted search on categorical fields, and sorting by text fields. For fields that need both word-level search and exact-match/faceting capabilities, use a multi-analyzer mapping combining `lucene.standard` for search and `lucene.keyword` for facets and sort.
