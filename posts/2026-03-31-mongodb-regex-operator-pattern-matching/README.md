# How to Use $regex for Pattern Matching in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $regex, Pattern Matching, Query, String

Description: Learn how to use MongoDB's $regex operator for pattern-based string queries, including case-insensitive matching, anchors, and index optimization.

---

## What Is $regex

The `$regex` operator matches string fields against a regular expression pattern. It is useful for prefix searches, contains checks, pattern validation, and more.

Two syntaxes are supported:

```javascript
// Using $regex operator
{ field: { $regex: /pattern/, $options: "i" } }

// Using $regex with string pattern
{ field: { $regex: "pattern", $options: "i" } }

// Using JavaScript regex directly (shorthand)
{ field: /pattern/ }
```

## Basic Examples

```javascript
// Find users whose name contains "john" (case-insensitive)
db.users.find({ name: { $regex: /john/i } })

// Equivalent with $options
db.users.find({ name: { $regex: "john", $options: "i" } })
```

## Anchored Patterns (Prefix and Suffix)

Anchoring your regex dramatically improves performance because MongoDB can use an index for prefix patterns:

```javascript
// Starts with "admin" (index-friendly)
db.users.find({ email: { $regex: /^admin/ } })

// Ends with "@example.com"
db.users.find({ email: { $regex: /@example\.com$/ } })
```

Prefix anchors (`^`) can use a regular ascending index. Non-anchored patterns require a full scan.

## Available $options Flags

| Flag | Description |
|------|-------------|
| `i` | Case-insensitive matching |
| `m` | Multiline - `^` and `$` match line boundaries |
| `s` | Dot-all - `.` matches newline characters |
| `x` | Extended - ignores whitespace in the pattern |

```javascript
// Case-insensitive, multiline
db.logs.find({ message: { $regex: "^error", $options: "im" } })
```

## Index Support for $regex

For index-backed regex queries, use anchored patterns on indexed string fields:

```javascript
await db.collection("users").createIndex({ username: 1 });

// Uses index efficiently (prefix anchor)
db.users.find({ username: /^alice/ })

// Does NOT efficiently use the index (no anchor)
db.users.find({ username: /alice/ })
```

For full-text search across many fields, consider a `$text` index instead.

## Negating a Regex with $not

```javascript
// Find emails that do NOT end with @internal.example.com
db.users.find({ email: { $not: /\@internal\.example\.com$/ } })
```

## $regex in $in Arrays

```javascript
// Match multiple patterns
db.users.find({
  email: { $in: [/@gmail\.com$/, /@yahoo\.com$/, /@outlook\.com$/] }
})
```

## Escaping Special Characters

Escape regex metacharacters when matching literal strings:

```javascript
// Match the literal string "user.name" (dot must be escaped)
db.docs.find({ field: { $regex: "user\\.name" } })
```

## $regex in Aggregation

```javascript
db.products.aggregate([
  {
    $match: {
      name: { $regex: /^Premium/i }
    }
  },
  { $project: { name: 1, price: 1 } }
])
```

## Common Mistakes

- Using unanchored patterns on large collections without a text index - causes full collection scans.
- Forgetting to escape regex special characters like `.`, `*`, `+`, `?`, `(`, `)`.
- Using case-insensitive flag `i` on indexed fields without a collation index - `$options: "i"` disables index usage.

## Summary

MongoDB's `$regex` operator matches string fields against regular expression patterns. Use anchored patterns (starting with `^`) to leverage indexes for prefix searches. For case-insensitive searches across large collections, consider a text index or collation-based index rather than `$regex` with the `i` flag, which cannot use a standard index.
