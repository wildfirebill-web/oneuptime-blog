# How to Use $match Early in the Pipeline for Better Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Performance, Query Optimization, Pipeline

Description: Learn how placing $match stages early in MongoDB aggregation pipelines reduces documents processed and dramatically improves query speed.

---

## Why $match Placement Matters

MongoDB aggregation pipelines process documents stage by stage. Each stage receives all documents output by the previous stage. If you filter documents late in the pipeline, every earlier stage must process the full collection, consuming CPU and memory unnecessarily.

Placing `$match` as early as possible reduces the working set before expensive operations like `$lookup`, `$group`, or `$sort` run.

## The Performance Impact

Consider a collection with 10 million orders. Without early filtering:

```javascript
// Slow: $group processes all 10 million documents
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $match: { total: { $gt: 1000 } } }
])
```

With early filtering:

```javascript
// Fast: $match reduces documents before $group
db.orders.aggregate([
  { $match: { status: "completed", createdAt: { $gte: ISODate("2024-01-01") } } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $match: { total: { $gt: 1000 } } }
])
```

The second pipeline only groups completed orders from 2024 onward - potentially processing 10x fewer documents.

## Index Usage and $match

When `$match` is the first stage, MongoDB can use indexes to satisfy it - just like a `find()` query. This is the most important optimization.

```javascript
// Create a compound index that matches your $match fields
db.orders.createIndex({ status: 1, createdAt: -1 })

// This $match will use the index
db.orders.aggregate([
  { $match: { status: "completed", createdAt: { $gte: ISODate("2024-01-01") } } },
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
  }},
  { $unwind: "$customer" },
  { $group: { _id: "$customer.region", revenue: { $sum: "$amount" } } }
])
```

Verify index usage with `explain()`:

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "completed", createdAt: { $gte: ISODate("2024-01-01") } } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
])
```

Look for `IXSCAN` in the winning plan - that confirms index usage.

## Splitting $match Stages

You can have multiple `$match` stages. MongoDB's query optimizer may merge adjacent `$match` stages, but it is still good practice to filter as early as possible at each logical step.

```javascript
// Pre-filter, then post-$lookup filter
db.orders.aggregate([
  // Stage 1: filter early using an index
  { $match: { status: "completed" } },

  // Stage 2: join with customers
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
  }},
  { $unwind: "$customer" },

  // Stage 3: filter on joined data (cannot be pushed earlier)
  { $match: { "customer.country": "US" } },

  { $group: { _id: "$customer.state", revenue: { $sum: "$amount" } } }
])
```

## Common Anti-Patterns

Avoid these patterns that prevent early filtering:

```javascript
// ANTI-PATTERN: $addFields before $match
// Forces the engine to add a field to ALL documents before filtering
db.orders.aggregate([
  { $addFields: { year: { $year: "$createdAt" } } },
  { $match: { year: 2024 } }
])

// BETTER: Use date operators directly in $match
db.orders.aggregate([
  { $match: {
      createdAt: {
        $gte: ISODate("2024-01-01"),
        $lt: ISODate("2025-01-01")
      }
  }},
  { $addFields: { year: { $year: "$createdAt" } } }
])
```

## Measuring Improvement

Use `explain("executionStats")` to compare before and after:

```javascript
const result = db.orders.explain("executionStats").aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$customerId", count: { $sum: 1 } } }
])

// Check nReturned, totalDocsExamined, executionTimeMillis
print(result.stages[0].$cursor.executionStats)
```

Key metrics to watch:
- `totalDocsExamined` - should match or be close to `nReturned` when using indexes
- `executionTimeMillis` - overall pipeline time
- `nReturned` from the `$match` stage - how many docs pass the filter

## Summary

Placing `$match` as the first stage of an aggregation pipeline allows MongoDB to use indexes and minimize the number of documents passed to subsequent stages. Always filter on indexed fields first, use `explain()` to verify index usage, and avoid computed fields in `$match` when the same filter can be expressed against raw document fields.
