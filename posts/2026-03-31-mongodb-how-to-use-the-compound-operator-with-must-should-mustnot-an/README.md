# How to Use the compound Operator in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Compound Query, Search

Description: Learn how to use the compound operator in MongoDB Atlas Search to combine must, should, mustNot, and filter clauses for complex search logic with relevance scoring.

---

## Overview

The `compound` operator in MongoDB Atlas Search combines multiple search operators using boolean logic. It is equivalent to a boolean query in Elasticsearch or Lucene. The four clauses - `must`, `should`, `mustNot`, and `filter` - give you precise control over which documents match, which are boosted, and which are excluded.

## The Four Clauses Explained

| Clause | Documents must match? | Affects score? |
|--------|-----------------------|----------------|
| `must` | Yes - all must match | Yes |
| `should` | No - but boosts score if matched | Yes |
| `mustNot` | Documents matching are excluded | No |
| `filter` | Yes - but does not affect score | No |

## Basic compound Query

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
            equals: {
              path: "inStock",
              value: true
            }
          }
        ]
      }
    }
  },
  { $limit: 10 },
  { $project: { title: 1, price: 1, score: { $meta: "searchScore" } } }
])
```

This returns in-stock laptops that are not refurbished.

## Using should to Boost Relevance

`should` clauses do not filter documents but increase the score for matching documents:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "headphones",
              path: ["title", "description"]
            }
          }
        ],
        should: [
          {
            // Boost documents with "wireless" in title
            text: {
              query: "wireless",
              path: "title",
              score: { boost: { value: 2 } }
            }
          },
          {
            // Boost documents with high ratings
            range: {
              path: "rating",
              gte: 4.5,
              score: { boost: { value: 1.5 } }
            }
          },
          {
            // Boost in-stock items slightly
            equals: {
              path: "inStock",
              value: true,
              score: { boost: { value: 1.2 } }
            }
          }
        ],
        mustNot: [
          {
            text: {
              query: "discontinued",
              path: "status"
            }
          }
        ]
      }
    }
  },
  {
    $project: {
      title: 1,
      rating: 1,
      inStock: 1,
      score: { $meta: "searchScore" }
    }
  },
  {
    $sort: { score: { $meta: "searchScore" } }
  },
  { $limit: 10 }
])
```

## minimumShouldMatch

Require at least N `should` clauses to match for a document to be included:

```javascript
db.articles.aggregate([
  {
    $search: {
      compound: {
        should: [
          { text: { query: "mongodb", path: "content" } },
          { text: { query: "performance", path: "content" } },
          { text: { query: "indexing", path: "content" } }
        ],
        minimumShouldMatch: 2  // at least 2 of the 3 should clauses must match
      }
    }
  }
])
```

## Nesting compound Operators

Compound operators can be nested for complex logic:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            compound: {
              should: [
                { text: { query: "laptop", path: "title" } },
                { text: { query: "notebook", path: "title" } },
                { text: { query: "ultrabook", path: "title" } }
              ],
              minimumShouldMatch: 1
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
        ],
        mustNot: [
          {
            text: {
              query: "gaming",
              path: "category"
            }
          }
        ]
      }
    }
  },
  { $limit: 10 }
])
```

## filter vs must: When to Use Each

Use `filter` when you want to narrow results based on structured data without affecting relevance score:

```javascript
// Correct: price filter does not dilute relevance score
{
  compound: {
    must: [
      { text: { query: "laptop", path: "title" } }  // affects score
    ],
    filter: [
      { range: { path: "price", gte: 500, lte: 1500 } }  // no score effect
    ]
  }
}
```

Use `must` for search terms where relevance matters:

```javascript
// Both terms must match and both affect score
{
  compound: {
    must: [
      { text: { query: "wireless", path: "title" } },
      { text: { query: "headphones", path: "title" } }
    ]
  }
}
```

## Combining with Post-Search Aggregation

After `$search`, use standard aggregation stages:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          { text: { query: "laptop", path: "title" } }
        ],
        filter: [
          { equals: { path: "inStock", value: true } }
        ]
      }
    }
  },
  // Post-search aggregation - runs on search results
  {
    $facet: {
      results: [
        { $limit: 10 },
        { $project: { title: 1, price: 1 } }
      ],
      priceRanges: [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 500, 1000, 2000, 5000],
            default: "5000+"
          }
        }
      ]
    }
  }
])
```

## Summary

The `compound` operator is the most flexible and powerful Atlas Search operator. Use `must` for required search terms that affect scoring, `filter` for structured constraints like price and inventory status that should not affect score, `should` for optional terms that boost relevance, and `mustNot` to exclude matching documents entirely. Nesting `compound` operators and using `minimumShouldMatch` enables sophisticated multi-condition search logic while maintaining precise control over relevance ranking.
