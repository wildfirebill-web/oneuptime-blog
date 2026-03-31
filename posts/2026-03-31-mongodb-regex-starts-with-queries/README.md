# How to Use Regex for Starts-With Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Regex, Query, Index, String

Description: Learn how to use regex for starts-with queries in MongoDB with the $regex operator and anchored patterns that can leverage prefix indexes.

---

## Introduction

A starts-with query finds all documents where a string field begins with a given prefix. In MongoDB, this is achieved using a regex with the `^` anchor. When written correctly, starts-with regex queries can use a B-tree index on the field, making them significantly faster than general-purpose regex scans.

## Basic Starts-With Query

Use the `^` anchor to match strings that start with a prefix:

```javascript
db.users.find({ username: /^admin/ })
```

This finds all documents where `username` starts with "admin".

Using the `$regex` operator with `$options`:

```javascript
db.users.find({
  username: { $regex: "^admin" }
})
```

Both forms are equivalent. The `/^pattern/` shorthand is more concise.

## Index Usage for Starts-With Queries

Anchored prefix patterns are the only type of regex query that can use a standard B-tree index. Create an index on the field:

```javascript
db.users.createIndex({ username: 1 })
```

Verify the index is used:

```javascript
db.users.find({ username: /^admin/ }).explain("executionStats")
```

In the output, look for:
```text
"stage": "IXSCAN"
"indexBounds": { "username": ["[\"admin\", \"admin\")", "[\"admiO\", \"admiO\")"] }
```

MongoDB converts the anchored prefix into a range scan over the index, which is very efficient.

## Starts-With with a Variable Prefix

When the prefix is a variable from user input, escape any special regex characters first:

```javascript
function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}

const userInput = "user.name+test";
const safePrefix = escapeRegex(userInput);
db.users.find({ username: new RegExp("^" + safePrefix) });
```

Without escaping, characters like `.` or `+` in the input would be interpreted as regex metacharacters.

## Starts-With on Nested Fields

```javascript
db.products.find({ "metadata.sku": /^ELEC-/ })
```

Ensure an index exists on the nested field path:

```javascript
db.products.createIndex({ "metadata.sku": 1 })
```

## Case-Insensitive Starts-With

For case-insensitive starts-with queries, use the `i` flag:

```javascript
db.users.find({ username: /^admin/i })
```

Note: case-insensitive regex (`i` flag) does NOT use a standard B-tree index. It always triggers a collection scan. Use a case-insensitive collation index instead if performance matters:

```javascript
db.users.createIndex(
  { username: 1 },
  { collation: { locale: "en", strength: 2 } }
)

db.users.find({ username: /^admin/ }).collation({ locale: "en", strength: 2 })
```

## Aggregation Pipeline Starts-With

Use `$regexMatch` in aggregation to filter or project based on starts-with logic:

```javascript
db.users.aggregate([
  {
    $match: {
      $expr: {
        $regexMatch: {
          input: "$username",
          regex: "^admin"
        }
      }
    }
  }
])
```

## Summary

Starts-with queries in MongoDB use the `^` anchor in a regex pattern. They are the only regex type that can leverage a B-tree index, making them efficient for large collections. For user-provided input, always escape special characters before building the regex. Case-insensitive starts-with queries require a collation index to avoid collection scans. The `$regexMatch` operator provides the same capability inside aggregation pipelines.
