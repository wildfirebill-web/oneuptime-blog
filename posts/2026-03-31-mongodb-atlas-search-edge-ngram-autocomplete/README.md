# How to Use Edge N-Gram Tokenizer for Autocomplete in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Autocomplete, Tokenizer, N-Gram

Description: Learn how to implement autocomplete search in MongoDB Atlas Search using the edgeGram tokenizer to match prefix tokens for real-time search-as-you-type functionality.

---

## What Is the Edge N-Gram Tokenizer?

The edge N-gram (edgeGram) tokenizer generates tokens anchored to the beginning of each word. For the input `"mongodb"`, it produces tokens of increasing length starting from the left edge: `"m"`, `"mo"`, `"mon"`, `"mong"`, `"mongo"`, `"mongod"`, `"mongodb"`.

This enables autocomplete: as a user types `"mon"`, documents indexed with edge N-grams for `"mongodb"` will match because `"mon"` is a valid token.

## Built-In Autocomplete Field Type

For simple cases, Atlas Search provides a dedicated `autocomplete` field type that handles edge N-grams automatically:

```javascript
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": {
        "type": "autocomplete",
        "analyzer": "lucene.standard",
        "tokenization": "edgeGram",
        "minGrams": 2,
        "maxGrams": 15
      }
    }
  }
}
```

Query with the `autocomplete` operator:

```javascript
db.products.aggregate([
  {
    $search: {
      index: "products_search",
      autocomplete: {
        query: "mon",
        path: "title",
        tokenOrder: "sequential"
      }
    }
  },
  { $project: { title: 1, score: { $meta: "searchScore" } } },
  { $sort: { score: -1 } },
  { $limit: 8 }
])
```

## Custom edgeGram Analyzer

For more control, build a custom analyzer using the `edgeGram` tokenizer:

```javascript
{
  "analyzers": [
    {
      "name": "autocomplete_analyzer",
      "tokenizer": {
        "type": "edgeGram",
        "minGram": 2,
        "maxGram": 20
      },
      "tokenFilters": [
        { "type": "lowercase" }
      ]
    }
  ],
  "mappings": {
    "fields": {
      "name": {
        "type": "string",
        "analyzer": "autocomplete_analyzer",
        "searchAnalyzer": "lucene.standard"
      }
    }
  }
}
```

The `searchAnalyzer` differs from the index analyzer here - index documents with edge N-grams, but search with standard tokenization. This ensures the query `"mongodb at"` finds documents starting with those words without generating N-grams at search time.

## Implementing a Search-as-You-Type API

```javascript
async function autocomplete(query, limit = 8) {
  if (!query || query.length < 2) return [];

  return db.collection("products").aggregate([
    {
      $search: {
        index: "products_autocomplete",
        autocomplete: {
          query: query.toLowerCase(),
          path: "name",
          fuzzy: { maxEdits: 1 }   // tolerate 1 typo
        }
      }
    },
    { $limit: limit },
    {
      $project: {
        name: 1,
        category: 1,
        score: { $meta: "searchScore" }
      }
    }
  ]).toArray();
}
```

The `fuzzy` option with `maxEdits: 1` handles minor typos in autocomplete queries.

## Configuring minGrams and maxGrams

```text
minGrams: minimum token length generated
maxGrams: maximum token length generated

For "laptop":
minGrams=2, maxGrams=10 -> ["la", "lap", "lapt", "lapto", "laptop"]
minGrams=1, maxGrams=10 -> ["l", "la", "lap", "lapt", "lapto", "laptop"]
```

Smaller `minGrams` values produce more tokens and larger index sizes but match shorter prefixes. A `minGrams` of 2 is recommended for most autocomplete use cases to avoid irrelevant single-character matches.

## Performance Considerations

Edge N-gram indexes are larger than standard indexes because each document generates multiple tokens. To keep performance high:

```javascript
// Limit autocomplete fields - only index fields used for autocomplete
{
  "mappings": {
    "dynamic": false,   // do NOT index all fields automatically
    "fields": {
      "name":        { "type": "autocomplete", "minGrams": 2, "maxGrams": 15 },
      "description": { "type": "string", "analyzer": "lucene.standard" }
    }
  }
}
```

Only apply the `autocomplete` type to fields that users actually type into. Index other fields with `lucene.standard` for regular search.

## Combining Autocomplete with Filters

```javascript
db.products.aggregate([
  {
    $search: {
      index: "products_search",
      compound: {
        must: [
          {
            autocomplete: {
              query: "wire",
              path: "name"
            }
          }
        ],
        filter: [
          {
            text: {
              query: "electronics",
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

## Summary

Atlas Search's edgeGram tokenizer powers autocomplete by indexing progressively longer prefixes anchored to word starts. For most use cases, use the built-in `autocomplete` field type with `tokenization: "edgeGram"`. For fine-grained control, define a custom analyzer with the `edgeGram` tokenizer and a standard `searchAnalyzer`. Set `minGrams: 2` and `maxGrams` between 15-20 to balance index size against matching flexibility. Add `fuzzy` matching to handle typos in partial queries.
