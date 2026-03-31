# How to Create a Multikey Index for Array Fields in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Multikey Index, Arrays, Database

Description: Learn how to create multikey indexes in MongoDB to efficiently query documents based on values within array fields, with examples and important constraints.

---

## Overview

When you create an index on a field that contains an array, MongoDB automatically creates a multikey index. Instead of indexing the array as a whole, MongoDB indexes each individual element of the array as a separate index entry. This allows queries that search for specific array values to use the index efficiently.

## Creating a Multikey Index

The syntax for creating a multikey index is identical to creating a regular index - MongoDB detects the array automatically:

```javascript
db.products.createIndex({ tags: 1 })
```

If any document in the `products` collection has an array value for `tags`, MongoDB creates a multikey index automatically.

## Querying with a Multikey Index

The index supports equality matches, `$in`, `$elemMatch`, and range queries on array elements:

```javascript
// Equality match on an array element
db.products.find({ tags: "electronics" })

// $in query
db.products.find({ tags: { $in: ["sale", "clearance"] } })

// $elemMatch for conditions on a single element
db.orders.find({
  items: { $elemMatch: { price: { $gt: 50 }, category: "tools" } }
})
```

## Indexing Arrays of Embedded Documents

Multikey indexes also work on arrays of embedded documents using dot notation:

```javascript
db.orders.createIndex({ "items.productId": 1 })
```

This indexes the `productId` field within every element of the `items` array:

```javascript
// This query can use the multikey index
db.orders.find({ "items.productId": ObjectId("...") })
```

## Compound Multikey Indexes

A compound index can include at most one array field. If two fields in a compound index are arrays, MongoDB returns an error:

```javascript
// Valid - only one array field (tags) in the compound index
db.products.createIndex({ category: 1, tags: 1 })

// This will fail if both 'tags' and 'colors' are arrays in the same document
// db.products.createIndex({ tags: 1, colors: 1 }) // Error!
```

## Checking if an Index is Multikey

Use `explain()` to confirm multikey index usage:

```javascript
db.products.find({ tags: "electronics" }).explain()
```

Look for `"isMultiKey": true` in the `queryPlanner.winningPlan.inputStage` section.

## Multikey Indexes and Array Bounds

For range queries on array elements, MongoDB uses index bounds to limit the scan range. However, compound multikey index bounds may be less precise than single-field index bounds for certain queries.

```javascript
// After creating: db.inventory.createIndex({ "specs.weight": 1 })
db.inventory.find({ "specs.weight": { $gt: 10, $lt: 50 } })
```

## Performance Considerations

Multikey indexes have some important characteristics:

```text
- Each array element creates a separate index entry (larger index size)
- Write operations (insert/update) update all index entries for modified arrays
- Very large arrays can significantly increase index write overhead
- For arrays with hundreds of elements per document, consider whether a different data model is more efficient
```

## Sparse Multikey Indexes

Create a sparse multikey index to exclude documents where the array field is missing:

```javascript
db.posts.createIndex({ tags: 1 }, { sparse: true })
```

This only indexes documents that have the `tags` field, reducing index size when many documents lack the field.

## Summary

MongoDB multikey indexes enable efficient querying on array fields by indexing each array element individually. They are created automatically when you index an array-valued field and support equality, range, and `$elemMatch` queries. Key constraints include the limit of one array field per compound index. For arrays with many elements, weigh the query performance gains against the increased index size and write overhead before creating multikey indexes.
