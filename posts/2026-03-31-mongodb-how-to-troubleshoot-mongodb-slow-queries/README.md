# How to Troubleshoot MongoDB Slow Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Slow Queries, Performance, Indexes, Explain

Description: Learn a step-by-step process to identify, analyze, and fix slow MongoDB queries using the profiler, explain plans, and index optimization.

---

## Step 1: Identify Slow Queries

Enable the MongoDB slow query log:

```javascript
// Profile all queries slower than 100ms
db.setProfilingLevel(1, { slowms: 100 })

// Check the current profiling level
db.getProfilingStatus()
// { was: 1, slowms: 100, sampleRate: 1.0 }
```

View slow query results:

```javascript
// Most recent slow queries
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()

// Slowest queries in the last hour
db.system.profile.find({
  ts: { $gte: new Date(Date.now() - 3600000) }
}).sort({ millis: -1 }).limit(20)

// Slowest queries by namespace
db.system.profile.aggregate([
  { $match: { ts: { $gte: new Date(Date.now() - 3600000) } } },
  { $group: {
      _id: "$ns",
      avgMs: { $avg: "$millis" },
      maxMs: { $max: "$millis" },
      count: { $sum: 1 }
  }},
  { $sort: { avgMs: -1 } }
])
```

## Step 2: Analyze with explain()

For a suspected slow query, run `explain("executionStats")`:

```javascript
db.orders.explain("executionStats").find({
  customerId: "cust-123",
  status: "pending"
})
```

Key fields in the output:

```text
executionStats:
  totalDocsExamined: 1524891  <- HIGH = full or near-full scan
  nReturned: 12               <- actual results
  totalKeysExamined: 12       <- should be close to nReturned for index use
  executionTimeMillis: 4823   <- query time in ms

winningPlan.stage:
  "COLLSCAN"  <- no index used - almost always needs fixing
  "IXSCAN"    <- using an index - good
  "SORT"      <- blocking in-memory sort - may need index
  "FETCH"     <- loading documents after index scan - normal
```

## Step 3: Interpret COLLSCAN vs IXSCAN

```javascript
// This plan is a problem - scanning all 1.5M documents
{
  "executionStats": {
    "totalDocsExamined": 1524891,
    "nReturned": 12,
    "executionTimeMillis": 4823
  },
  "winningPlan": {
    "stage": "COLLSCAN"
  }
}

// This plan is healthy - using an index
{
  "executionStats": {
    "totalDocsExamined": 12,
    "totalKeysExamined": 12,
    "nReturned": 12,
    "executionTimeMillis": 2
  },
  "winningPlan": {
    "stage": "FETCH",
    "inputStage": { "stage": "IXSCAN", "indexName": "customerId_1_status_1" }
  }
}
```

## Step 4: Add Appropriate Indexes

Based on the query pattern from explain output:

```javascript
// Query: { customerId: "...", status: "pending" }
// Add a compound index on both filter fields
db.orders.createIndex({ customerId: 1, status: 1 })

// Query: { status: "pending" } sorted by { createdAt: -1 }
// Include sort field in index
db.orders.createIndex({ status: 1, createdAt: -1 })

// Verify the index is used
db.orders.explain("executionStats").find(
  { customerId: "cust-123", status: "pending" }
).sort({ createdAt: -1 })
// Should now show IXSCAN instead of COLLSCAN
```

## Step 5: Fix Blocking Sort Stages

```javascript
// Slow: in-memory sort on large result set
db.orders.find({ status: "pending" }).sort({ createdAt: -1 })

// Check explain output
db.orders.explain("executionStats").find(
  { status: "pending" }
).sort({ createdAt: -1 })
// Look for "SORT" stage with large memLimit usage

// Fix: index that supports both filter and sort
db.orders.createIndex({ status: 1, createdAt: -1 })
// Now MongoDB reads documents in sorted order - no in-memory sort
```

## Step 6: Use $limit to Reduce Work

```javascript
// Without limit - scans everything
db.events.find({ userId: "user123" }).sort({ timestamp: -1 })

// With limit - stops after 20 results
db.events.find({ userId: "user123" }).sort({ timestamp: -1 }).limit(20)
// Much faster, even without an ideal index
```

## Step 7: Avoid $ne and $nin on Unindexed Fields

```javascript
// SLOW: negation queries often require full scans
db.users.find({ status: { $ne: "deleted" } })

// BETTER: use positive conditions instead
db.users.find({ status: { $in: ["active", "pending", "suspended"] } })

// Or model the data differently:
// Instead of filtering out "deleted", mark active users
db.users.createIndex({ isActive: 1 })
db.users.find({ isActive: true })
```

## Step 8: Profile Aggregation Pipelines

```javascript
// Add explain to aggregation
db.orders.explain("executionStats").aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
])

// Look at the first stage in the pipeline
// If it shows COLLSCAN, add an index on { status: 1 }
```

## Continuous Monitoring

```javascript
// Keep profiling on with a 200ms threshold in production
db.setProfilingLevel(1, { slowms: 200 })

// Atlas: use Performance Advisor for automated index recommendations
// Atlas automatically identifies slow queries and suggests indexes
```

## Summary

Troubleshoot slow MongoDB queries by enabling the profiler with a slow query threshold, using `explain("executionStats")` to identify `COLLSCAN` plans and high `totalDocsExamined` counts, then adding compound indexes that cover both filter and sort fields. Fix blocking sort stages by including sort keys in indexes, use `$limit` to reduce work when pagination is appropriate, and avoid negation operators on unindexed fields.
