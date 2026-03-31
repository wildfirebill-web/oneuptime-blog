# How to Use Regex for Contains Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Regex, Query, Text Search, String

Description: Learn how to use regex for contains queries in MongoDB to find documents where a field includes a substring, and when to use text indexes instead.

---

## Introduction

A contains query finds documents where a string field includes a given substring anywhere in its value. In MongoDB, this uses a regex pattern without anchors. Unlike starts-with patterns, contains regex queries cannot use a B-tree index and always perform a collection scan. For full-text search at scale, a text index is more appropriate.

## Basic Contains Query

Match any document where a field contains a substring:

```javascript
db.articles.find({ title: /mongodb/ })
```

Using the `$regex` operator:

```javascript
db.articles.find({ title: { $regex: "mongodb" } })
```

Both return documents where `title` contains "mongodb" anywhere in the string.

## Contains with Variable Input

Escape special characters when building a contains pattern from user input:

```javascript
function buildContainsRegex(input) {
  const escaped = input.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
  return new RegExp(escaped);
}

const searchTerm = "node.js";
db.articles.find({ title: buildContainsRegex(searchTerm) });
```

Without escaping, the `.` in "node.js" would match any character.

## Case-Insensitive Contains

```javascript
db.articles.find({ title: /mongodb/i })
```

The `i` flag makes the match case-insensitive. Like all non-anchored regex, this triggers a full collection scan.

## Performance Implications

Contains queries always scan the entire collection (or index entries, if using a collection scan on an indexed field):

```javascript
db.articles.find({ title: /mongodb/ }).explain("executionStats")
// stage: "COLLSCAN" - full collection scan
```

For large collections, this is slow. Options to improve performance:

1. **Text index** - for word-based full-text search:

```javascript
db.articles.createIndex({ title: "text", content: "text" })

db.articles.find({ $text: { $search: "mongodb" } })
```

Text indexes support stemming, stop words, and relevance scoring, but only match whole words.

2. **Atlas Search** - for advanced substring and fuzzy matching on MongoDB Atlas:

```javascript
db.articles.aggregate([
  {
    $search: {
      index: "default",
      wildcard: {
        query: "*mongodb*",
        path: "title",
        allowAnalyzedField: true
      }
    }
  }
])
```

## Contains Query in Aggregation

Use `$regexMatch` in an aggregation pipeline:

```javascript
db.articles.aggregate([
  {
    $match: {
      $expr: {
        $regexMatch: {
          input: "$title",
          regex: "mongodb",
          options: "i"
        }
      }
    }
  },
  {
    $project: {
      title: 1,
      hasKeyword: { $regexMatch: { input: "$title", regex: "mongodb" } }
    }
  }
])
```

## Limiting Scan Impact

If you must use contains regex on a large collection, add a selective `$match` on an indexed field earlier in the pipeline to reduce the number of documents the regex must scan:

```javascript
db.articles.find({
  category: "Database",
  title: /mongodb/i
})
```

With an index on `category`, MongoDB narrows the candidate set before applying the regex.

## Summary

Contains queries in MongoDB use unanchored regex patterns that always trigger a collection scan. For small collections or infrequent queries this is acceptable, but for large collections consider a text index for word-level matching or MongoDB Atlas Search for substring and fuzzy matching. When using contains regex in aggregation, apply a selective `$match` on an indexed field first to limit the document set the regex must scan.
