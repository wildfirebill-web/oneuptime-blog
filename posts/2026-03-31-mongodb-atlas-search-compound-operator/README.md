# How to Use the compound Operator with must, should, mustNot, and filter

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Compound, Full-Text Search, Atlas

Description: Learn how to use the Atlas Search compound operator to combine must, should, mustNot, and filter clauses for complex multi-condition search queries.

---

The `compound` operator in MongoDB Atlas Search lets you combine multiple search conditions with different scoring behaviors. It is the Atlas Search equivalent of a boolean query in Elasticsearch and is the primary way to build complex search logic.

## The Four Clauses

| Clause | Behavior |
|--------|----------|
| `must` | Document must match - contributes to score |
| `should` | Document should match - contributes to score but not required |
| `mustNot` | Document must NOT match - does not contribute to score |
| `filter` | Document must match - does NOT contribute to score |

## Basic must and should Example

Find articles that must contain "mongodb" and should contain "performance":

```javascript
db.articles.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "mongodb",
              path: "title"
            }
          }
        ],
        should: [
          {
            text: {
              query: "performance optimization",
              path: "body"
            }
          }
        ]
      }
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

## Using mustNot to Exclude Results

Exclude articles tagged as "sponsored":

```javascript
db.articles.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "kubernetes deployment",
              path: ["title", "body"]
            }
          }
        ],
        mustNot: [
          {
            text: {
              query: "sponsored",
              path: "tags"
            }
          }
        ]
      }
    }
  }
])
```

## Using filter to Restrict Without Scoring

The `filter` clause applies conditions without affecting relevance scores. Use it for category or date filters:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "running shoes",
              path: ["name", "description"]
            }
          }
        ],
        filter: [
          {
            range: {
              path: "price",
              gte: 50,
              lte: 200
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
    $project: {
      name: 1,
      price: 1,
      score: { $meta: "searchScore" }
    }
  }
])
```

## Combining All Four Clauses

A product search that requires a keyword, boosts reviews, excludes discontinued items, and filters by category:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "bluetooth speaker",
              path: ["name", "description"]
            }
          }
        ],
        should: [
          {
            range: {
              path: "reviewScore",
              gte: 4.0,
              score: { boost: { value: 2 } }
            }
          }
        ],
        mustNot: [
          {
            equals: {
              path: "status",
              value: "discontinued"
            }
          }
        ],
        filter: [
          {
            equals: {
              path: "category",
              value: "electronics"
            }
          }
        ]
      }
    }
  },
  {
    $project: {
      name: 1,
      reviewScore: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $sort: { score: -1 } },
  { $limit: 10 }
])
```

## minimumShouldMatch

Require at least N `should` clauses to match:

```javascript
db.articles.aggregate([
  {
    $search: {
      compound: {
        should: [
          { text: { query: "python", path: "tags" } },
          { text: { query: "machine learning", path: "tags" } },
          { text: { query: "neural network", path: "tags" } }
        ],
        minimumShouldMatch: 2
      }
    }
  }
])
```

## Summary

The `compound` operator is the foundation of complex Atlas Search queries. Use `must` for required terms, `filter` for non-scoring restrictions like category or price, `should` for relevance boosting, and `mustNot` to exclude unwanted documents.

