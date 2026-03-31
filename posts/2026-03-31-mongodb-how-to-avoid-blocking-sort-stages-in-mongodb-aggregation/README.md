# How to Avoid Blocking Sort Stages in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Sorting, Performance, Index

Description: Learn how to eliminate blocking sort stages in MongoDB aggregation pipelines by using indexes and rewriting pipelines for better throughput.

---

## What Is a Blocking Sort Stage?

A blocking sort stage in MongoDB aggregation means the `$sort` operation must collect all input documents into memory before it can output even a single result. This is called a "blocking" operation because the next stage cannot begin until sorting is complete.

This becomes a problem when:
- The collection is large
- The sort does not use an index
- The sort exceeds the 100MB memory limit (requires `allowDiskUse: true`)

## Detecting Blocking Sorts

Use `explain()` to identify blocking sorts:

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "shipped" } },
  { $sort: { createdAt: -1 } },
  { $limit: 20 }
])
```

In the explain output, look for `SORT` in the winning plan. If you see `SORT` without `IXSCAN` feeding it, the sort is blocking. A non-blocking sort will show `IXSCAN` with the sort direction matching the index direction.

## Using Indexes to Avoid Blocking Sorts

The most effective solution is an index that covers both the `$match` filter and the `$sort` order:

```javascript
// Create an index on status and createdAt
db.orders.createIndex({ status: 1, createdAt: -1 })

// Now this pipeline uses an index scan in sort order - no blocking sort
db.orders.aggregate([
  { $match: { status: "shipped" } },
  { $sort: { createdAt: -1 } },
  { $limit: 20 }
])
```

MongoDB can use the index to return documents already in sorted order, eliminating the need for an in-memory sort stage entirely.

## The $sort + $limit Optimization

When `$sort` is immediately followed by `$limit`, MongoDB applies a special optimization called a top-K sort. Instead of sorting all documents, it maintains a heap of the K smallest (or largest) elements:

```javascript
// MongoDB optimizes this into a top-20 sort - much more efficient
db.events.aggregate([
  { $match: { type: "pageview" } },
  { $sort: { timestamp: -1 } },
  { $limit: 20 }
])
```

Even without an index, a top-K sort is far more memory efficient than a full sort.

## Rewriting Pipelines to Push $sort Later

Sometimes you can defer sorting to after filtering, reducing the number of documents being sorted:

```javascript
// AVOID: Sorting before filtering
db.logs.aggregate([
  { $sort: { timestamp: -1 } },
  { $match: { level: "ERROR" } },
  { $limit: 100 }
])

// BETTER: Filter first, then sort a smaller set
db.logs.aggregate([
  { $match: { level: "ERROR" } },
  { $sort: { timestamp: -1 } },
  { $limit: 100 }
])
```

## Handling allowDiskUse

For large sorts that cannot use an index, you may need `allowDiskUse`:

```javascript
db.analytics.aggregate(
  [
    { $group: { _id: "$userId", events: { $sum: 1 } } },
    { $sort: { events: -1 } }
  ],
  { allowDiskUse: true }
)
```

This spills to disk when the 100MB memory limit is reached. It is slower than an in-memory sort but prevents `MongoServerError: Exceeded memory limit`. Use this as a fallback, not a primary strategy.

## Using $setWindowFields Instead of $sort + $group

In some cases, `$setWindowFields` can replace patterns that require sorting every document:

```javascript
// Instead of sorting to find cumulative totals
db.sales.aggregate([
  { $setWindowFields: {
      partitionBy: "$region",
      sortBy: { date: 1 },
      output: {
        runningTotal: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        }
      }
  }}
])
```

`$setWindowFields` uses an internal sort but can leverage indexes and often performs better than a standalone `$sort` followed by complex grouping logic.

## Compound Index Design for Sort

When designing indexes for aggregation, match the sort key order and direction:

```javascript
// Query filters on userId, sorts by createdAt descending
db.orders.createIndex({ userId: 1, createdAt: -1 })

db.orders.aggregate([
  { $match: { userId: "user123" } },
  { $sort: { createdAt: -1 } },
  { $project: { amount: 1, createdAt: 1 } }
])
```

The compound index `{ userId: 1, createdAt: -1 }` serves both the equality filter on `userId` and the range scan in sort order.

## Summary

Blocking sort stages are one of the most common performance bottlenecks in MongoDB aggregation. Eliminate them by creating indexes that match your sort fields, leveraging the `$sort + $limit` top-K optimization, filtering documents before sorting, and using `allowDiskUse` only when necessary. Always verify with `explain()` that your sort uses an index scan rather than an in-memory sort.
