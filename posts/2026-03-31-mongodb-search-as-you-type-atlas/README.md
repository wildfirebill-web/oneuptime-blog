# How to Implement Search-As-You-Type with Atlas Search in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Autocomplete, Search-As-You-Type, Full-Text Search

Description: Build a search-as-you-type experience in MongoDB Atlas Search using the autocomplete operator with edge n-gram tokenization for instant results.

---

## What Is Search-As-You-Type?

Search-as-you-type (also called typeahead) shows relevant results as the user types each character. In MongoDB Atlas Search, this is implemented using the `autocomplete` operator, which uses an edge n-gram analyzer to match partial words from the beginning of tokens.

## Creating an Autocomplete Index

Create an Atlas Search index with `autocomplete` field mappings:

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": [
        {
          "type": "autocomplete",
          "analyzer": "lucene.standard",
          "tokenization": "edgeGram",
          "minGrams": 2,
          "maxGrams": 15
        },
        {
          "type": "string"
        }
      ],
      "description": {
        "type": "autocomplete",
        "analyzer": "lucene.standard",
        "tokenization": "edgeGram",
        "minGrams": 2,
        "maxGrams": 10
      }
    }
  }
}
```

## Basic Search-As-You-Type Query

```javascript
// Called with each keystroke: query = "wire" (user has typed 4 chars)
db.products.aggregate([
  {
    $search: {
      index: "autocomplete_index",
      autocomplete: {
        query: "wire",
        path: "title",
        tokenOrder: "sequential"
      }
    }
  },
  { $limit: 5 },
  { $project: { title: 1, _id: 0 } }
])
```

## Multi-Field Autocomplete

Search across multiple fields simultaneously:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        should: [
          {
            autocomplete: {
              query: userInput,
              path: "title",
              score: { boost: { value: 3 } }
            }
          },
          {
            autocomplete: {
              query: userInput,
              path: "brand",
              score: { boost: { value: 2 } }
            }
          },
          {
            autocomplete: {
              query: userInput,
              path: "category"
            }
          }
        ]
      }
    }
  },
  { $limit: 8 },
  { $project: { title: 1, brand: 1, category: 1 } }
])
```

## Adding Relevance Scores to Results

```javascript
db.products.aggregate([
  {
    $search: {
      autocomplete: {
        query: "lapt",
        path: "title",
        tokenOrder: "sequential"
      }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $sort: { score: -1 } },
  { $limit: 10 }
])
```

## Filtering Autocomplete Results

Combine autocomplete with filters:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            autocomplete: {
              query: "head",
              path: "title"
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
  { $limit: 5 }
])
```

## Node.js Example

```javascript
async function searchAsYouType(db, query) {
  if (query.length < 2) return [];

  return db.collection("products").aggregate([
    {
      $search: {
        autocomplete: {
          query: query,
          path: "title",
          tokenOrder: "sequential"
        }
      }
    },
    { $limit: 8 },
    { $project: { title: 1, category: 1, price: 1, _id: 1 } }
  ]).toArray();
}
```

## Tokenization Options

| Tokenization | When to Use |
|---|---|
| `edgeGram` | Match from start of each word |
| `rightEdgeGram` | Match from end of each word |
| `nGram` | Match anywhere in a word |

`edgeGram` is most common for search-as-you-type as it matches how users type words from left to right.

## Performance Tips

- Set `minGrams: 2` to avoid too many index entries for single characters
- Keep `maxGrams` reasonable (10-15) to limit index size
- Add a `$limit` stage immediately after `$search` to reduce data processed by subsequent stages
- Use `compound.filter` for category/facet filtering since filter clauses skip scoring

## Summary

Atlas Search's `autocomplete` operator with `edgeGram` tokenization powers search-as-you-type experiences. Create an index with autocomplete field mappings, use `tokenOrder: "sequential"` for phrase matching, and combine with `compound.filter` for faceted filtering. Keep `minGrams` at 2 and apply `$limit` early in the pipeline for fast response times.
