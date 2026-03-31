# How to Split Strings in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, String, Aggregation, Split, Array, Pipeline

Description: Learn how to split strings in MongoDB aggregation pipelines using the $split operator to parse delimited fields into arrays.

---

## Introduction

The `$split` aggregation operator divides a string into an array of substrings based on a delimiter. It is useful for parsing CSV-like fields, extracting path segments, splitting tags stored as comma-separated strings, and processing structured text data embedded in string fields.

## Basic $split Usage

```javascript
db.products.aggregate([
  {
    $project: {
      sku: 1,
      parts: { $split: ["$sku", "-"] }
    }
  }
]);
```

For a SKU like `"LAPTOP-001-BLK"`, this produces `["LAPTOP", "001", "BLK"]`.

## Parsing CSV Tags

Convert a comma-separated tag string to an array:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      tagArray: {
        $split: ["$tagsCsv", ","]
      }
    }
  }
]);
```

For `"mongodb,database,nosql"`, the result is `["mongodb", "database", "nosql"]`.

## Trimming Split Results

Combine `$split` with `$map` and `$trim` to clean each element:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      cleanTags: {
        $map: {
          input: { $split: ["$tagsCsv", ","] },
          as: "tag",
          in: { $trim: { input: "$$tag" } }
        }
      }
    }
  }
]);
```

This handles inputs like `"mongodb, database , nosql"` by removing surrounding spaces from each tag.

## Extracting URL Path Segments

Parse a URL path into its segments:

```javascript
db.logs.aggregate([
  {
    $project: {
      path: 1,
      segments: { $split: ["$path", "/"] }
    }
  }
]);
```

For `/api/v2/users/123`, this produces `["", "api", "v2", "users", "123"]`.

## Converting Split Strings to Tags and Unwinding

Split and unwind to normalize tags into individual documents:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      tags: { $split: ["$tagsCsv", ","] }
    }
  },
  { $unwind: "$tags" },
  {
    $project: {
      title: 1,
      tag: { $toLower: { $trim: { input: "$tags" } } }
    }
  },
  { $group: { _id: "$tag", articleCount: { $sum: 1 } } },
  { $sort: { articleCount: -1 } }
]);
```

## Using $arrayElemAt After Split

Extract a specific part of a split result:

```javascript
db.products.aggregate([
  {
    $project: {
      category: {
        $arrayElemAt: [{ $split: ["$sku", "-"] }, 0]
      }
    }
  }
]);
```

This extracts the first segment of the SKU as the category code.

## Summary

The `$split` operator in MongoDB aggregation pipelines converts delimited strings into arrays, enabling you to parse structured text fields without application-level preprocessing. Combining `$split` with `$map`, `$trim`, and `$unwind` gives you a complete toolkit for normalizing and analyzing string data directly in the database.
