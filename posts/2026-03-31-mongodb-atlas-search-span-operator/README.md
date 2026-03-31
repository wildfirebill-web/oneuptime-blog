# How to Use the span Operator for Token Proximity in Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Span, Proximity, Atlas

Description: Learn how to use the Atlas Search span operator to match documents where search terms appear within a defined token distance of each other.

---

The `span` operator in MongoDB Atlas Search finds documents where terms appear within a specific number of tokens (words) of each other. Unlike `phrase` which requires adjacent words, `span` lets you define how far apart terms can be while still considering them related.

## span Sub-operators

The `span` operator has several nested clauses:

| Sub-operator | Description |
|---|---|
| `spanFirst` | Term appears within N tokens from the start |
| `spanNear` | Terms appear within N tokens of each other |
| `spanOr` | Matches any of the given span queries |
| `spanNot` | Excludes documents matching the inner span |

## Basic spanNear Example

Find documents where "database" and "performance" appear within 5 tokens of each other:

```javascript
db.articles.aggregate([
  {
    $search: {
      span: {
        spanNear: {
          clauses: [
            {
              span: {
                spanTerm: {
                  path: "body",
                  query: "database"
                }
              }
            },
            {
              span: {
                spanTerm: {
                  path: "body",
                  query: "performance"
                }
              }
            }
          ],
          slop: 5,
          inOrder: false
        }
      }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" }
    }
  }
])
```

`inOrder: false` means the terms can appear in either order.

## Ordered Proximity Search

Require "query" to appear before "optimization" within 3 words:

```javascript
db.articles.aggregate([
  {
    $search: {
      span: {
        spanNear: {
          clauses: [
            {
              span: {
                spanTerm: { path: "body", query: "query" }
              }
            },
            {
              span: {
                spanTerm: { path: "body", query: "optimization" }
              }
            }
          ],
          slop: 3,
          inOrder: true
        }
      }
    }
  },
  {
    $project: { title: 1 }
  }
])
```

## spanFirst: Term Near Document Start

Find documents where "introduction" appears in the first 10 tokens:

```javascript
db.articles.aggregate([
  {
    $search: {
      span: {
        spanFirst: {
          query: {
            span: {
              spanTerm: {
                path: "body",
                query: "introduction"
              }
            }
          },
          end: 10
        }
      }
    }
  },
  {
    $project: { title: 1 }
  }
])
```

## spanOr: Multiple Proximity Patterns

Match documents satisfying any of several proximity patterns:

```javascript
db.articles.aggregate([
  {
    $search: {
      span: {
        spanOr: {
          clauses: [
            {
              span: {
                spanNear: {
                  clauses: [
                    { span: { spanTerm: { path: "body", query: "index" } } },
                    { span: { spanTerm: { path: "body", query: "performance" } } }
                  ],
                  slop: 4,
                  inOrder: false
                }
              }
            },
            {
              span: {
                spanNear: {
                  clauses: [
                    { span: { spanTerm: { path: "body", query: "query" } } },
                    { span: { spanTerm: { path: "body", query: "speed" } } }
                  ],
                  slop: 4,
                  inOrder: false
                }
              }
            }
          ]
        }
      }
    }
  },
  {
    $project: { title: 1, score: { $meta: "searchScore" } }
  }
])
```

## When to Use span vs phrase

```text
Operator | Use case
---------|----------------------------------------------------
phrase   | Exact or near-exact word sequences (slop <= 2)
span     | Precise positional control, ordered proximity, startof-doc matching
```

## Summary

The `span` operator provides fine-grained positional control over how terms relate to each other within a document. Use `spanNear` for proximity matching, `spanFirst` for terms near the beginning of a field, and `spanOr` when multiple positional patterns are acceptable. It is most useful for legal text, technical documentation, or any domain where word proximity carries semantic meaning.

