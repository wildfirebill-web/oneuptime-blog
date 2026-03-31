# How to Find Documents with Array Fields That Are Empty in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Array, Filter

Description: Learn how to query MongoDB for documents where an array field is empty, null, or missing, with index-friendly patterns and Python examples.

---

## Three Meanings of "Empty Array"

When working with array fields, "empty" can mean three different things:

1. The field holds an empty array: `{ tags: [] }`
2. The field is explicitly set to `null`: `{ tags: null }`
3. The field is missing from the document entirely

Each situation requires a slightly different query.

## Finding Documents with an Empty Array

To match only documents where the array exists and has zero elements, query for the empty array literal:

```javascript
db.products.find({ tags: [] })
```

This is an exact equality match against the empty array.

## Finding Documents Where the Array Is Empty, Null, or Missing

Use `$in` to combine all three conditions in a single query:

```javascript
db.products.find({
  tags: { $in: [null, []] }
})
```

Or use `$or` for clarity:

```javascript
db.products.find({
  $or: [
    { tags: { $exists: false } },
    { tags: null },
    { tags: [] }
  ]
})
```

## Using $size to Match Zero-Length Arrays

The `$size` operator matches arrays of an exact length. Size zero means empty:

```javascript
db.products.find({ tags: { $size: 0 } })
```

Note that `$size` does not match documents where the field is null or missing - it only matches documents where the field is an array of the given length.

## Combining with $exists and $not

To find documents that have the field but the array is empty (excluding missing documents):

```javascript
db.products.find({
  tags: { $exists: true, $size: 0 }
})
```

To find documents with a non-empty array:

```javascript
db.products.find({
  tags: { $exists: true, $not: { $size: 0 } }
})
```

## Using the Aggregation Pipeline

The aggregation pipeline gives you more expressive filtering with `$match` and `$expr`:

```javascript
db.products.aggregate([
  {
    $match: {
      $expr: {
        $or: [
          { $eq: [{ $type: "$tags" }, "missing"] },
          { $eq: ["$tags", null] },
          { $eq: [{ $size: { $ifNull: ["$tags", []] } }, 0] }
        ]
      }
    }
  }
])
```

## Python Example

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["shop"]

# Documents with empty, null, or missing tags
empty_tags = list(db.products.find({"tags": {"$in": [None, []]}}))

# Documents with exactly an empty array
strictly_empty = list(db.products.find({"tags": []}))

print(f"Empty/null/missing: {len(empty_tags)}")
print(f"Strictly empty array: {len(strictly_empty)}")
```

## Indexing Considerations

A standard single-field index on an array field will index the array's elements but will also index the empty array value `[]`. This means queries for empty arrays can use the index:

```javascript
db.products.createIndex({ tags: 1 })
```

Run `explain()` to confirm the query plan uses the index:

```javascript
db.products.find({ tags: [] }).explain("executionStats")
```

## Summary

Use `{ tags: [] }` for an exact empty array match. Use `{ tags: { $size: 0 } }` to match only zero-length arrays (excluding null/missing). Combine `$in: [null, []]` or `$or` to catch all three empty states together. Add a regular index on the array field to keep these queries efficient as the collection grows.
