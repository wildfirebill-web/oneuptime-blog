# How to Search for Strings Using Regular Expressions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Regex, Query, String, Index, Pattern Matching

Description: Learn how to search for strings using regular expressions in MongoDB with $regex, case-insensitive flags, and index-compatible prefix patterns.

---

## Introduction

MongoDB supports regular expression queries using the `$regex` operator. You can use regex to search for patterns within string fields, perform case-insensitive matching, and find partial matches. Understanding how regex queries interact with indexes is critical for writing performant queries.

## Basic Regex Query

Find all users whose email contains "gmail":

```javascript
db.users.find({ email: { $regex: "gmail" } });
```

Or using the `/pattern/` shorthand:

```javascript
db.users.find({ email: /gmail/ });
```

## Case-Insensitive Search

Use the `i` option for case-insensitive matching:

```javascript
db.users.find({ username: { $regex: "alice", $options: "i" } });
// Matches: "alice", "Alice", "ALICE", "aLiCe"
```

Or with regex literal:

```javascript
db.users.find({ username: /alice/i });
```

## Anchored Prefix Queries (Index-Friendly)

Queries anchored at the start of a string (`^`) can use an index:

```javascript
// This can use an index on the "username" field
db.users.find({ username: /^ali/ });

// Case-insensitive prefix - cannot use standard index
db.users.find({ username: /^ali/i });
```

For case-insensitive prefix search with index support, use a collation index instead of regex with the `i` flag.

## Matching Specific Patterns

Find phone numbers matching a specific format:

```javascript
db.contacts.find({
  phone: { $regex: "^\\+1-[0-9]{3}-[0-9]{3}-[0-9]{4}$" }
});
```

Find SKUs with a specific pattern:

```javascript
db.products.find({ sku: /^[A-Z]{3}-\d{3}$/ });
```

## Using $regex in Aggregation

Use `$regexMatch` in aggregation pipelines:

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
  }
]);
```

Or filter in a `$project`:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      hasMongoDB: {
        $regexMatch: { input: "$title", regex: "mongodb", options: "i" }
      }
    }
  }
]);
```

## Performance Considerations

Regex queries without a caret anchor (`^`) perform a full collection scan. To avoid this:

1. Use text indexes for full-text search:

```javascript
db.articles.createIndex({ title: "text", content: "text" });
db.articles.find({ $text: { $search: "mongodb" } });
```

2. Use prefix anchoring when possible (`^pattern`).

3. For case-insensitive indexed lookups, use collation:

```javascript
db.users.createIndex({ username: 1 }, { collation: { locale: "en", strength: 2 } });
db.users.find({ username: "alice" }).collation({ locale: "en", strength: 2 });
```

## Summary

MongoDB regex queries with `$regex` provide flexible string pattern matching. Anchored prefix queries (`^pattern`) are index-compatible and efficient. Case-insensitive searches with the `i` flag require full collection scans unless backed by a collation index. For full-text search across document content, text indexes with `$text` are significantly more efficient than regex queries.
