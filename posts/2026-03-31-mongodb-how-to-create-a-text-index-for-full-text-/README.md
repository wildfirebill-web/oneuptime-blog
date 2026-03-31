# How to Create a Text Index for Full-Text Search in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Full-Text Search, Text Index, Database

Description: Learn how to create text indexes in MongoDB to enable full-text search on string fields, including multi-field indexes, language configuration, and $text queries.

---

## Overview

MongoDB text indexes support full-text search queries on string content. A text index tokenizes and stems the indexed text, removing stop words and enabling language-aware searching. Unlike regular indexes that match exact values, text indexes support word-based searches, relevance scoring, and phrase matching.

## Creating a Single Field Text Index

Use the `"text"` index type instead of `1` or `-1`:

```javascript
db.articles.createIndex({ body: "text" })
```

After creating the index, use `$text` with `$search` to query it:

```javascript
db.articles.find({ $text: { $search: "mongodb aggregation" } })
```

This searches for documents containing either "mongodb" or "aggregation" (OR by default).

## Creating a Text Index on Multiple Fields

Index multiple string fields in a single text index. A collection can have only one text index:

```javascript
db.products.createIndex({
  name: "text",
  description: "text",
  tags: "text"
})
```

You can assign different weights to fields to influence relevance scoring:

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
)
```

## Creating a Wildcard Text Index

Index all string fields in documents using `$**`:

```javascript
db.documents.createIndex({ "$**": "text" })
```

This is convenient but creates a larger index and may not perform as well as targeted indexes.

## Text Search Query Operations

Phrase search using quotes:

```javascript
db.articles.find({ $text: { $search: "\"full text search\"" } })
```

Exclude a term with a minus sign:

```javascript
db.articles.find({ $text: { $search: "mongodb -relational" } })
```

Search for multiple terms (all must match - AND):

```javascript
db.articles.find({ $text: { $search: "\"mongodb\" \"aggregation pipeline\"" } })
```

## Sorting by Text Relevance Score

Use the `$meta` projection to get and sort by relevance score:

```javascript
db.articles.find(
  { $text: { $search: "mongodb performance" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

## Language Configuration

Text indexes support language-specific stemming and stop word lists:

```javascript
db.articles.createIndex(
  { body: "text" },
  { default_language: "french" }
)
```

Override language at query time:

```javascript
db.articles.find({
  $text: {
    $search: "recherche",
    $language: "french"
  }
})
```

For documents in multiple languages, store the language in a field and use it during indexing:

```javascript
db.articles.createIndex(
  { body: "text" },
  { language_override: "lang" }
)
```

## Case Insensitivity and Diacritics

By default, text search is case-insensitive and diacritic-insensitive. The `$caseSensitive` and `$diacriticSensitive` options can override this:

```javascript
db.articles.find({
  $text: {
    $search: "MongoDB",
    $caseSensitive: true
  }
})
```

## Checking Text Index Status

View the text index details:

```javascript
db.articles.getIndexes()
```

## Summary

MongoDB text indexes enable full-text search directly in the database without requiring an external search engine for basic use cases. They support word tokenization, stemming, stop word removal, phrase matching, term exclusion, and relevance scoring. Configure field weights for ranking, specify languages for proper stemming, and use `$meta: "textScore"` for relevance-sorted results. For advanced search requirements like faceting or fuzzy matching, consider Atlas Search or Elasticsearch.
