# How to Use $match Early in the Pipeline for Better Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Performance, Index, Pipeline

Description: Learn why placing $match early in MongoDB aggregation pipelines improves performance through index usage and reduced document processing across stages.

---

## The Role of $match in Aggregation Performance

In MongoDB aggregation, every document that enters a stage consumes CPU and memory. When `$match` is placed later in the pipeline, all documents from earlier stages must be processed before any filtering occurs. Moving `$match` to the front of the pipeline allows MongoDB to leverage indexes and eliminate irrelevant documents before any other work is done.

## How $match Leverages Indexes

When `$match` is the first stage in a pipeline, MongoDB's query optimizer can use collection indexes to satisfy the filter criteria - exactly as it would for a standard `find()` query. This means only matching documents are loaded and passed to subsequent stages.

```javascript
// Create a supporting index
db.orders.createIndex({ status: 1, createdAt: -1 });

// $match first - index is used, only "shipped" orders proceed
db.orders.aggregate([
  { $match: { status: "shipped", createdAt: { $gte: new Date("2025-01-01") } } },
  { $group: { _id: "$customerId", count: { $sum: 1 } } }
]);
```

Run `explain()` to confirm index use:

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "shipped" } },
  { $group: { _id: "$customerId", count: { $sum: 1 } } }
]);
```

Look for `IXSCAN` in the winning plan to confirm index usage.

## Incorrect Stage Ordering

Placing transformation stages before `$match` breaks index eligibility and forces full collection scans.

```javascript
// Bad: $addFields before $match prevents index use on computed field
db.orders.aggregate([
  { $addFields: { year: { $year: "$createdAt" } } },
  { $match: { year: 2025 } }   // Cannot use an index here
]);

// Good: filter on the raw field directly
db.orders.aggregate([
  { $match: { createdAt: { $gte: new Date("2025-01-01"), $lt: new Date("2026-01-01") } } },
  { $addFields: { year: { $year: "$createdAt" } } }
]);
```

## Multiple $match Stages

You can and should use multiple `$match` stages when filtering depends on earlier computation. The optimizer merges consecutive `$match` stages automatically.

```javascript
db.sales.aggregate([
  { $match: { region: "US" } },               // Hits index
  { $lookup: {
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product"
  }},
  { $unwind: "$product" },
  { $match: { "product.category": "Electronics" } }  // Post-lookup filter
]);
```

## Combining $match with $sort

For queries requiring both filtering and sorting, placing `$match` before `$sort` allows a compound index to serve both operations together.

```javascript
db.logs.createIndex({ level: 1, timestamp: -1 });

db.logs.aggregate([
  { $match: { level: "error" } },
  { $sort: { timestamp: -1 } },
  { $limit: 100 }
]);
```

This compound index handles the match and sort in a single efficient pass.

## Pushing $match Through $project

MongoDB's aggregation optimizer can automatically push a `$match` stage through a preceding `$project` stage if the match criteria reference the original document fields - not computed fields. However, it is safer and clearer to write the `$match` before `$project` explicitly.

```javascript
// Explicit and clear: always preferred
db.users.aggregate([
  { $match: { country: "US", active: true } },
  { $project: { name: 1, email: 1, country: 1 } }
]);
```

## Summary

Placing `$match` as early as possible in a MongoDB aggregation pipeline is one of the most effective performance optimizations available. It enables index usage, reduces the number of documents flowing through subsequent stages, and lowers both CPU and memory consumption. Always check the pipeline execution plan with `explain()` to verify that indexes are being used and that no unnecessary documents are processed downstream.
