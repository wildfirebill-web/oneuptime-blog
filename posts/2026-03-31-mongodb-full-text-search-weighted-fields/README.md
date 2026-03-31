# How to Implement Full-Text Search with Weighted Fields in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Text Search, Index, Relevance, Query

Description: Learn how to implement full-text search with weighted fields in MongoDB using text indexes and $text queries to boost relevance scoring by field importance.

---

## Overview

MongoDB's built-in text search supports field weighting, allowing matches in more important fields (like a title) to score higher than matches in less important fields (like a body or description). This improves result relevance without a dedicated search engine.

## Creating a Weighted Text Index

Assign weights when creating the text index. Higher weight means greater relevance contribution.

```javascript
db.articles.createIndex(
  {
    title: "text",
    summary: "text",
    body: "text"
  },
  {
    weights: {
      title: 10,
      summary: 5,
      body: 1
    },
    name: "article_text_index"
  }
);
```

A match in `title` contributes 10x more to the score than a match in `body`.

## Querying with $text

```javascript
db.articles.find(
  { $text: { $search: "kubernetes monitoring" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } });
```

The `$meta: "textScore"` projection adds the computed relevance score to each result, and sorting by it returns the most relevant first.

## Phrase Search

Wrap phrases in escaped quotes to require exact phrase matching.

```javascript
db.articles.find(
  { $text: { $search: "\"open source monitoring\"" } },
  { score: { $meta: "textScore" } }
);
```

## Excluding Terms

Prefix a term with `-` to exclude documents containing it.

```javascript
db.articles.find(
  { $text: { $search: "kubernetes -docker" } },
  { score: { $meta: "textScore" } }
);
```

## Combining Text Search with Other Filters

```javascript
db.articles.find(
  {
    $text: { $search: "alerting" },
    published: true,
    category: "devops"
  },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } });
```

## Using in Aggregation

Use `$match` with `$text` and then project the score.

```javascript
db.articles.aggregate([
  { $match: { $text: { $search: "incident response" } } },
  { $addFields: { score: { $meta: "textScore" } } },
  { $sort: { score: -1 } },
  { $limit: 10 },
  { $project: { title: 1, summary: 1, score: 1 } }
]);
```

## Checking Index Weights

View the existing text index configuration.

```javascript
db.articles.getIndexes().filter(
  idx => idx.key._fts === "text"
);
```

## Limitations

- Only one text index is allowed per collection.
- Text indexes do not support case-sensitive matching; all searches are case-insensitive.
- For advanced relevance tuning, consider Atlas Search with Lucene-backed scoring.

## Summary

MongoDB text indexes with field weights improve full-text search relevance by scoring matches in high-priority fields (like title) more heavily than lower-priority fields. Create the index with a `weights` map, retrieve scores using `$meta: "textScore"`, and sort by score to return the most relevant documents first.
