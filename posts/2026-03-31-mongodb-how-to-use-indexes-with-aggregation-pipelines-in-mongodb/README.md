# How to Use Indexes with Aggregation Pipelines in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Indexing, Query Optimization, Performance

Description: Learn how to optimize MongoDB aggregation pipelines with indexes by placing $match and $sort early and using covered pipelines to eliminate FETCH stages.

---

## How Aggregation Pipelines Use Indexes

MongoDB aggregation pipelines can leverage indexes, but only if certain conditions are met. The key rule: indexes are used only at the beginning of a pipeline by `$match`, `$sort`, and `$geoNear` stages. Once a transformation stage runs (like `$project`, `$addFields`, or `$group`), the optimizer can no longer use indexes for stages that come after.

## The Golden Rule: $match and $sort First

```javascript
// BAD: $project before $match blocks index use
db.orders.aggregate([
  { $project: { status: 1, amount: 1 } },  // transformation first
  { $match: { status: "pending" } }         // index can't be used here
]);

// GOOD: $match first uses index
db.orders.aggregate([
  { $match: { status: "pending" } },        // uses index on status
  { $project: { status: 1, amount: 1 } }
]);
```

## Verifying Index Use with explain()

```javascript
// Always verify with explain
const result = db.orders.explain("executionStats").aggregate([
  { $match: { status: "pending", region: "west" } },
  { $group: { _id: "$category", total: { $sum: "$amount" } } }
]);

// Look for IXSCAN in the $cursor stage of the explain output
// Red flag: COLLSCAN means no index is being used
```

## Example 1 - $match with Compound Index

```javascript
// Create a compound index matching your $match conditions
db.orders.createIndex({ status: 1, region: 1, createdAt: -1 });

// Pipeline that fully leverages the compound index
db.orders.aggregate([
  {
    $match: {
      status: "pending",           // uses index prefix
      region: "west",              // uses index prefix
      createdAt: { $gte: new Date("2026-01-01") }  // uses index range
    }
  },
  {
    $group: {
      _id: "$customerId",
      totalAmount: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalAmount: -1 } },
  { $limit: 10 }
]);
```

## Example 2 - $sort Using Index (No In-Memory Sort)

Place `$sort` before any transformation stages to use an index:

```javascript
// Create index matching sort order
db.events.createIndex({ userId: 1, timestamp: -1 });

// $sort immediately after $match uses the compound index
db.events.aggregate([
  { $match: { userId: "user-123" } },
  { $sort: { timestamp: -1 } },    // uses compound index - no in-memory sort
  { $limit: 50 },
  { $project: { eventType: 1, timestamp: 1, payload: 1 } }
]);

// If you must project before sort, the sort can't use an index
db.events.aggregate([
  { $match: { userId: "user-123" } },
  { $project: { type: "$eventType", ts: "$timestamp" } },
  { $sort: { ts: -1 } }  // in-memory sort - SORT stage in explain
]);
```

## Example 3 - Covered Aggregation Pipeline

A covered pipeline reads all needed data from the index without touching documents:

```javascript
// Index covering all fields needed by the pipeline
db.sales.createIndex({ region: 1, category: 1, amount: 1 });

// Covered pipeline - no FETCH stage needed
db.sales.aggregate([
  { $match: { region: "west" } },
  {
    $group: {
      _id: "$category",
      totalAmount: { $sum: "$amount" },
      count: { $sum: 1 }
    }
  }
]);
```

In the explain output, a covered pipeline shows IXSCAN without a following FETCH stage.

## Example 4 - $lookup with Index on Foreign Collection

```javascript
// Create index on the foreignField in the joined collection
db.customers.createIndex({ _id: 1 });           // usually exists already
db.customers.createIndex({ externalId: 1 });    // for non-_id joins

db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",        // uses _id index on customers
      as: "customer",
      pipeline: [
        { $project: { name: 1, tier: 1 } }  // limit data returned
      ]
    }
  }
]);
```

## Example 5 - $unwind and Index Recovery

After `$unwind`, you cannot use indexes from the original collection. Minimize data before unwinding:

```javascript
// BAD: unwind before match loses index opportunity
db.orders.aggregate([
  { $unwind: "$lineItems" },
  { $match: { "lineItems.category": "Electronics" } }  // COLLSCAN on unwound docs
]);

// GOOD: filter at document level first (if possible), then unwind
db.orders.aggregate([
  { $match: { status: "completed", "lineItems.category": "Electronics" } },  // index
  { $unwind: "$lineItems" },
  { $match: { "lineItems.category": "Electronics" } }  // filter after unwind
]);

// Create multikey index for array field filter
db.orders.createIndex({ "lineItems.category": 1, status: 1 });
```

## Example 6 - $geoNear Uses Geospatial Index

`$geoNear` must be the first stage and requires a 2dsphere or 2d index:

```javascript
db.locations.createIndex({ coordinates: "2dsphere" });

db.locations.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-122.4, 37.8] },
      distanceField: "distanceMeters",
      maxDistance: 5000,
      spherical: true,
      query: { category: "restaurant" }  // additional filter using index
    }
  },
  { $limit: 20 }
]);
```

## Pipeline Optimization Checklist

```text
1. Always start with $match using indexed fields
2. Put $sort before $limit for top-N queries
3. Use $limit early to reduce pipeline cardinality
4. Avoid $project before $match or $sort
5. Create compound indexes matching ($match fields, $sort fields)
6. Use covered pipelines when $group only needs indexed fields
7. Index foreignField in $lookup target collections
8. Use $match before $unwind to filter whole documents first
9. Verify with explain("executionStats") - look for IXSCAN not COLLSCAN
10. Watch for SORT stages in explain - they indicate missing sort index
```

## Summary

MongoDB aggregation pipelines use indexes primarily for `$match`, `$sort`, and `$geoNear` stages placed at the beginning of the pipeline. The optimizer pushes these stages into a `$cursor` phase that leverages the collection's indexes before any transformation stages run. To maximize index use: place `$match` with indexed fields first, put `$sort` immediately after `$match` before any projections, create compound indexes following the ESR rule, and verify all pipeline optimizations with `explain("executionStats")` to confirm IXSCAN stages and the absence of in-memory SORT stages.
