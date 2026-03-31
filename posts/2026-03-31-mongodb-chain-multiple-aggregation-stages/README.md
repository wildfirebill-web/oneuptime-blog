# How to Chain Multiple Aggregation Stages Efficiently in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline, Performance, Optimization

Description: Learn how to chain multiple MongoDB aggregation stages efficiently, applying ordering rules and optimization techniques to reduce memory and execution time.

---

A MongoDB aggregation pipeline is a sequence of stages - each stage transforms the documents flowing through it. The order and composition of stages directly determines performance. Poor stage ordering can force full collection scans and in-memory sorts on millions of documents.

## The Core Principle: Filter Early, Project Late

Place `$match` and `$limit` as early as possible to reduce the working set before heavy operations like `$group`, `$sort`, or `$lookup`.

```javascript
// Inefficient - groups all documents first
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $match: { total: { $gt: 1000 } } }
])

// Efficient - filters before grouping
db.orders.aggregate([
  { $match: { status: "completed", createdAt: { $gte: ISODate("2024-01-01") } } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $match: { total: { $gt: 1000 } } }
])
```

## Stage Ordering Best Practices

A well-ordered pipeline follows this general structure:

```text
$match       -> filter documents (uses indexes)
$sort        -> sort early if index-backed, otherwise after $match
$limit       -> reduce set before expensive stages
$lookup      -> join after reducing the left-side set
$unwind      -> flatten arrays after $lookup if needed
$group       -> aggregate the filtered, joined data
$project     -> reshape output (only after reducing)
$sort        -> final sort on computed fields
$skip/$limit -> pagination at the end
```

## Practical Multi-Stage Example

Report the top 5 customers by revenue in 2024 with order counts:

```javascript
db.orders.aggregate([
  // Stage 1: filter by time range and status - uses index
  {
    $match: {
      createdAt: { $gte: ISODate("2024-01-01"), $lt: ISODate("2025-01-01") },
      status: "completed"
    }
  },
  // Stage 2: group to compute per-customer totals
  {
    $group: {
      _id: "$customerId",
      totalRevenue: { $sum: "$amount" },
      orderCount:   { $sum: 1 }
    }
  },
  // Stage 3: sort by revenue descending
  { $sort: { totalRevenue: -1 } },
  // Stage 4: limit to top 5 before lookup
  { $limit: 5 },
  // Stage 5: enrich with customer details
  {
    $lookup: {
      from: "customers",
      localField: "_id",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" },
  // Stage 6: shape the final output
  {
    $project: {
      _id: 0,
      name: "$customer.name",
      email: "$customer.email",
      totalRevenue: 1,
      orderCount: 1
    }
  }
])
```

## Using $facet for Parallel Sub-Pipelines

When you need multiple aggregations over the same input in a single query, use `$facet`:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $facet: {
      summary: [
        { $group: { _id: null, total: { $sum: "$amount" }, count: { $sum: 1 } } }
      ],
      topRegions: [
        { $group: { _id: "$region", revenue: { $sum: "$amount" } } },
        { $sort: { revenue: -1 } },
        { $limit: 5 }
      ]
    }
  }
])
```

## Analyzing Pipeline Performance with explain

Always verify your pipeline uses indexes and avoids COLLSCAN:

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
])
```

Look for `IXSCAN` in the winning plan and check `totalDocsExamined` vs `totalDocsReturned`. A large ratio indicates the `$match` is not selective enough or is missing an index.

## Enabling allowDiskUse for Large Pipelines

If aggregation exceeds the 100 MB memory limit:

```javascript
db.orders.aggregate(pipeline, { allowDiskUse: true })
```

## Summary

Efficient stage chaining in MongoDB means filtering with `$match` and `$limit` first to minimize document flow, then performing joins and grouping on a reduced set, and projecting fields last. Use `explain()` to confirm index usage and `$facet` for parallel sub-pipelines when multiple views of the same data are needed.
