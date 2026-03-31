# How to Use $text for Full-Text Search Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $text, Full-Text Search, Text Index, Query

Description: Learn how to use MongoDB's $text operator with a text index to perform full-text search, filter by language, and sort results by relevance score.

---

## Prerequisites: Creating a Text Index

Before using `$text`, you must create a text index on the fields you want to search:

```javascript
// Single-field text index
await db.collection("articles").createIndex({ content: "text" });

// Multi-field text index
await db.collection("products").createIndex({
  name: "text",
  description: "text"
});

// Wildcard text index (all string fields)
await db.collection("docs").createIndex({ "$**": "text" });
```

A collection can have at most one text index.

## Basic $text Search

```javascript
// Find articles containing "mongodb" or "performance"
db.articles.find({ $text: { $search: "mongodb performance" } })
```

By default, MongoDB performs an OR search - documents containing any of the words are returned.

## Phrase Search

Wrap phrases in escaped quotes to require exact phrase matching:

```javascript
// Must contain the exact phrase "query optimization"
db.articles.find({
  $text: { $search: "\"query optimization\"" }
})
```

## Excluding Words with a Minus Sign

```javascript
// Contains "mongodb" but NOT "atlas"
db.articles.find({
  $text: { $search: "mongodb -atlas" }
})
```

## Combining Multiple Terms

```javascript
// Contains "index" AND ("performance" OR "optimization") but NOT "deprecated"
db.articles.find({
  $text: { $search: "\"index\" performance optimization -deprecated" }
})
```

## Sorting by Relevance Score

Use `$meta: "textScore"` to project and sort by relevance:

```javascript
db.articles.find(
  { $text: { $search: "mongodb sharding" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

Higher scores indicate better matches.

## Language-Specific Search

MongoDB supports language-specific stemming and stop words:

```javascript
// Search with French language rules
db.articles.find({
  $text: {
    $search: "gestion base données",
    $language: "french"
  }
})
```

Supported languages include `english`, `french`, `spanish`, `german`, `portuguese`, and more. Use `"none"` to disable stemming.

## Case and Diacritic Insensitivity

```javascript
// caseSensitive defaults to false
db.products.find({
  $text: {
    $search: "MongoDB",
    $caseSensitive: false,      // default
    $diacriticSensitive: false  // normalize accented characters
  }
})
```

## Combining $text with Other Filters

```javascript
db.products.find({
  $text: { $search: "laptop" },
  price: { $lte: 1000 },
  inStock: true
})
```

## Limitations of $text

- Only one `$text` expression is allowed per query.
- `$text` cannot be used in `$or` arrays.
- You cannot use a hint with a `$text` query.
- Aggregation `$match` with `$text` must be the first stage.

## Text Search in Aggregation

```javascript
db.articles.aggregate([
  { $match: { $text: { $search: "performance tuning" } } },
  { $project: { title: 1, score: { $meta: "textScore" } } },
  { $sort: { score: -1 } },
  { $limit: 10 }
])
```

## Common Mistakes

- Searching without creating a text index first - MongoDB returns an error.
- Expecting exact substring matching - `$text` uses word stemming, not substring search.
- Trying to use `$text` in an `$or` clause - this is not allowed.

## Summary

MongoDB's `$text` operator enables full-text search on collections with a text index. It supports phrase matching, word exclusion, relevance scoring, and language-specific stemming. For substring or pattern matching, use `$regex` instead. For more advanced search features like fuzzy matching and facets, consider MongoDB Atlas Search.
