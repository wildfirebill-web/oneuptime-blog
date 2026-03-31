# How to Use $count in MongoDB Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Count

Description: Learn how to use the $count stage in MongoDB aggregation to count documents passing through the pipeline and when to use it versus countDocuments() or $group.

---

## What Is the $count Stage?

The `$count` stage counts the number of documents that reach it in the aggregation pipeline and outputs a single document with the count under a specified field name. It is a concise alternative to using `$group` with `$sum: 1` when you only need a total count.

## Basic Syntax

```javascript
db.collection.aggregate([
  { $count: "<outputFieldName>" }
])
```

The output is a single document like `{ "<outputFieldName>": <number> }`.

## Example: Count All Documents

```javascript
db.orders.aggregate([
  { $count: "totalOrders" }
])
// Output: { totalOrders: 2547 }
```

## Example: Count After Filtering

```javascript
db.orders.aggregate([
  { $match: { status: "pending" } },
  { $count: "pendingCount" }
])
// Output: { pendingCount: 143 }
```

## Counting with Multiple Filters

```javascript
db.users.aggregate([
  {
    $match: {
      plan: "pro",
      createdAt: { $gte: new Date("2024-01-01") }
    }
  },
  { $count: "newProUsers" }
])
```

## $count vs $group for Counting

```javascript
// Using $count - simpler
db.events.aggregate([
  { $match: { type: "login" } },
  { $count: "loginEvents" }
])

// Equivalent using $group
db.events.aggregate([
  { $match: { type: "login" } },
  { $group: { _id: null, loginEvents: { $sum: 1 } } }
])
```

Use `$count` when you just need a single total. Use `$group` when you need per-category counts.

## Per-Group Counts with $group + $sum

For counting by category, `$count` alone is not enough - use `$group`:

```javascript
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } }
])
```

## $count vs countDocuments()

```javascript
// Simple count - no pipeline needed
const total = await db.collection("orders").countDocuments({ status: "pending" })

// Aggregation count - use when chained with other stages
const result = await db.collection("orders").aggregate([
  { $match: { status: "pending" } },
  { $unwind: "$items" },
  { $count: "totalPendingItems" }
]).toArray()
```

Use `countDocuments()` for simple counts. Use `$count` in aggregation when the count requires prior pipeline stages like `$unwind`, `$lookup`, or `$group`.

## Counting After $unwind

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  { $match: { "items.category": "electronics" } },
  { $count: "electronicLineItems" }
])
```

## Getting Count as Part of a Larger Result

Use `$facet` to include a count alongside other aggregation results.

```javascript
db.products.aggregate([
  {
    $facet: {
      data: [
        { $match: { category: "electronics" } },
        { $sort: { price: 1 } },
        { $limit: 20 }
      ],
      total: [
        { $match: { category: "electronics" } },
        { $count: "count" }
      ]
    }
  }
])
```

## Summary

The `$count` stage is the simplest way to count documents in a MongoDB aggregation pipeline. Place it after `$match`, `$unwind`, or other filtering stages to count documents that pass through. For per-group counts, use `$group` with `$sum: 1`. For simple queries without a pipeline, `countDocuments()` is more efficient.
