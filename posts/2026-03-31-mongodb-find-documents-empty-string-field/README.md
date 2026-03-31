# How to Find Documents Where a String Field Is Empty in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, String, Filter

Description: Learn how to query MongoDB for documents where a string field is empty, null, or missing, using exact match, regex, and aggregation operators.

---

## The Empty String Query

To find documents where a string field exists and equals an empty string, use a simple equality filter:

```javascript
db.users.find({ bio: "" })
```

This matches documents where `bio` is the empty string `""`. It does not match documents where `bio` is `null`, `undefined`, or absent.

## Combining Empty String, Null, and Missing

In practice you often want to treat all three states as "blank":

```javascript
db.users.find({
  $or: [
    { bio: "" },
    { bio: null },
    { bio: { $exists: false } }
  ]
})
```

A shorter version using `$in` handles both empty string and null:

```javascript
db.users.find({ bio: { $in: ["", null] } })
```

`$in` with `null` matches both `null` values and missing fields in MongoDB.

## Using a Regex for Whitespace-Only Strings

A field containing only spaces is technically non-empty, but semantically blank. Use a regex anchored at start and end:

```javascript
db.users.find({ bio: /^\s*$/ })
```

This matches `""`, `" "`, `"\t"`, `"\n"`, and similar whitespace-only strings.

## Using the Aggregation Pipeline for Complex Conditions

The `$expr` operator lets you combine string functions inside a `$match` stage:

```javascript
db.users.aggregate([
  {
    $match: {
      $expr: {
        $or: [
          { $eq: ["$bio", ""] },
          { $eq: ["$bio", null] },
          {
            $eq: [
              { $trim: { input: { $ifNull: ["$bio", ""] } } },
              ""
            ]
          }
        ]
      }
    }
  },
  { $project: { name: 1, bio: 1 } }
])
```

`$trim` strips leading and trailing whitespace before comparing, catching " " and "\t" cases.

## Finding Non-Empty Strings

The inverse query - documents where the string is present and not empty - is equally common:

```javascript
db.users.find({
  bio: { $exists: true, $ne: null, $nin: [""] }
})
```

Or with regex anchoring:

```javascript
db.users.find({ bio: /\S+/ })  // at least one non-whitespace character
```

## Python Example

```python
from pymongo import MongoClient
import re

client = MongoClient("mongodb://localhost:27017")
db = client["myapp"]

# All blank bio variants
blank_users = list(db.users.find(
    {"bio": {"$in": ["", None]}},
    {"name": 1, "bio": 1}
))

# Whitespace-only using regex
whitespace_users = list(db.users.find(
    {"bio": re.compile(r"^\s*$")},
    {"name": 1, "bio": 1}
))

print(f"Blank bio: {len(blank_users)}")
print(f"Whitespace bio: {len(whitespace_users)}")
```

## Index Usage

A standard index on the string field supports equality queries efficiently:

```javascript
db.users.createIndex({ bio: 1 })
```

Regex queries using a left-anchored pattern (`/^someprefix/`) can also use an index. The empty-string regex `^\s*$` is not left-anchored with a literal prefix, so it will perform a full index scan. For large collections, prefer the equality approach `{ bio: { $in: ["", null] } }` which is fully index-backed.

## Summary

Use `{ bio: "" }` for an exact empty string match. Use `{ bio: { $in: ["", null] } }` to catch both empty and null values in one query. Use the regex `/^\s*$/` to find whitespace-only strings. In the aggregation pipeline, combine `$ifNull` and `$trim` for the most thorough blank-value detection. Always ensure the field has an index to avoid collection scans on large datasets.
