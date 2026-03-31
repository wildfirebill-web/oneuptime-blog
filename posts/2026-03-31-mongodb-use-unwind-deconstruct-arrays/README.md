# How to Use $unwind to Deconstruct Arrays in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Array

Description: Learn how to use MongoDB's $unwind stage to deconstruct array fields into individual documents, enabling per-element aggregation and cross-stage array analysis.

---

## What Is the $unwind Stage?

The `$unwind` stage deconstructs an array field, outputting one document per array element. Each output document is identical to the input except the array field is replaced by a single element. If a document has an array with 5 elements, `$unwind` produces 5 output documents from that single input document.

## Basic Syntax

```javascript
// Shorthand
db.collection.aggregate([
  { $unwind: "$arrayField" }
])

// Full syntax with options
db.collection.aggregate([
  {
    $unwind: {
      path: "$arrayField",
      includeArrayIndex: "<indexFieldName>",
      preserveNullAndEmpty: <boolean>
    }
  }
])
```

## Example: Unwinding an Order's Items

```javascript
// Document:
// { _id: "order-1", items: [{ sku: "A", qty: 2 }, { sku: "B", qty: 1 }] }

db.orders.aggregate([
  { $unwind: "$items" }
])

// Output:
// { _id: "order-1", items: { sku: "A", qty: 2 } }
// { _id: "order-1", items: { sku: "B", qty: 1 } }
```

## Aggregating After $unwind

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.sku",
      totalQty: { $sum: "$items.qty" },
      revenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } }
    }
  },
  { $sort: { revenue: -1 } }
])
```

This computes revenue per SKU across all orders.

## Including the Array Index

```javascript
db.playlists.aggregate([
  {
    $unwind: {
      path: "$tracks",
      includeArrayIndex: "trackPosition"
    }
  }
])
// Each document now has a trackPosition field (0, 1, 2, ...)
```

## Preserving Empty Arrays and Missing Fields

By default, `$unwind` drops documents with null, missing, or empty array fields.

```javascript
db.orders.aggregate([
  {
    $unwind: {
      path: "$tags",
      preserveNullAndEmpty: true
    }
  }
])
// Documents with no tags field or empty tags: [] are kept
```

## Filtering After $unwind

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  { $match: { "items.category": "electronics" } },
  { $group: { _id: "$_id", electronicItems: { $push: "$items" } } }
])
```

## Re-Grouping After $unwind

A common pattern is to unwind, transform, and re-group.

```javascript
db.articles.aggregate([
  { $unwind: "$tags" },
  { $match: { tags: { $ne: "draft" } } },
  {
    $group: {
      _id: "$_id",
      title: { $first: "$title" },
      tags: { $push: "$tags" }
    }
  }
])
// Removes "draft" tag from all articles
```

## $unwind on Nested Arrays

```javascript
db.courses.aggregate([
  { $unwind: "$modules" },
  { $unwind: "$modules.lessons" }
])
// One document per lesson across all courses and modules
```

## Performance Considerations

`$unwind` can significantly increase document count. Always filter with `$match` before `$unwind` when possible to reduce the number of documents being exploded.

```javascript
// Better: filter before unwind
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $unwind: "$items" }
])
```

## Summary

The `$unwind` stage is essential for per-element analysis of array fields in MongoDB aggregation. It converts multi-element arrays into individual documents for grouping, filtering, and transformation. Use `preserveNullAndEmpty` to retain documents without the array field, and always apply `$match` before `$unwind` to minimize the pipeline workload.
