# How to Check if a Field Is an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Array, Type, Filter

Description: Learn how to check if a field is an array in MongoDB using the $type operator, $isArray aggregation expression, and related query patterns.

---

MongoDB documents can store mixed types - a field that is an array in one document might be a string or missing in another. Checking whether a field is an array lets you filter documents correctly and avoid type errors in aggregations.

## Use $type to Query for Array Fields

The `$type` operator matches documents where the specified field is of a given BSON type. Arrays have BSON type `4`:

```javascript
// Find documents where "tags" is an array
db.products.find({ tags: { $type: "array" } })

// Using numeric BSON type code
db.products.find({ tags: { $type: 4 } })
```

## Check for Array vs Non-Array

Find documents where a field exists but is NOT an array:

```javascript
db.products.find({
  tags: { $exists: true, $not: { $type: "array" } }
})
```

## Use $isArray in Aggregation

In aggregation pipelines, use `$isArray` to return a boolean or branch logic:

```javascript
db.products.aggregate([
  {
    $addFields: {
      tagsIsArray: { $isArray: "$tags" }
    }
  }
])
```

Filter documents where the field is an array using `$match` with `$expr`:

```javascript
db.products.aggregate([
  {
    $match: {
      $expr: { $isArray: "$tags" }
    }
  }
])
```

## Handle Mixed Types Safely

When processing data where a field might be an array or a scalar, use `$cond` to normalize it:

```javascript
db.products.aggregate([
  {
    $addFields: {
      tagsArray: {
        $cond: {
          if: { $isArray: "$tags" },
          then: "$tags",
          else: {
            $cond: {
              if: { $ne: ["$tags", null] },
              then: ["$tags"],  // Wrap scalar in array
              else: []
            }
          }
        }
      }
    }
  }
])
```

## Find Documents with an Array of a Specific Size

Combine `$type` with `$size` to find arrays with an exact element count:

```javascript
// Documents where "tags" is an array with exactly 3 elements
db.products.find({
  tags: { $type: "array", $size: 3 }
})
```

## Use $ifNull with Array Checks

Safely get the size of a field that might not be an array:

```javascript
db.products.aggregate([
  {
    $addFields: {
      tagCount: {
        $cond: {
          if: { $isArray: "$tags" },
          then: { $size: "$tags" },
          else: 0
        }
      }
    }
  }
])
```

This avoids the "the argument to $size must be an array" error that occurs when calling `$size` on a non-array value.

## Index Considerations

`$type` queries on array fields can use indexes, but performance depends on the index type and query selectivity. For large collections, create an index on the field and use covered queries where possible:

```javascript
db.products.createIndex({ tags: 1 })
db.products.find({ tags: { $type: "array" } }).explain("executionStats")
```

## Summary

Use `$type: "array"` (or `$type: 4`) in find queries to match documents where a field is an array. In aggregation pipelines, use `$isArray` for boolean checks and conditional logic. When a field has mixed types across documents, normalize it with `$cond` and `$isArray` before applying array-specific operators like `$size` or `$unwind`.
