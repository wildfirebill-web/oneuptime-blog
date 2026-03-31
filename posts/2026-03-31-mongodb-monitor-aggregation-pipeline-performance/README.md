# How to Monitor Aggregation Pipeline Performance in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Performance, Profiling, Index

Description: Learn how to profile and monitor aggregation pipeline performance in MongoDB using explain plans, the profiler, and Atlas metrics.

---

## Why Monitor Aggregation Performance

Aggregation pipelines can consume significant CPU and memory, especially when they involve large collections, multiple unwind stages, or blocking sorts. Identifying slow pipelines early prevents performance degradation in production.

## Using explain() on Aggregation Pipelines

The `explain` command reveals how MongoDB executes each pipeline stage:

```javascript
db.orders.explain("executionStats").aggregate([
    { $match: { status: "shipped", createdAt: { $gte: new Date("2024-01-01") } } },
    { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
    { $sort: { total: -1 } },
    { $limit: 10 }
]);
```

Key fields to look for in the output:

```javascript
// Look for these fields in executionStats
{
    "stages": [
        {
            "$cursor": {
                "executionStats": {
                    "nReturned": 5000,
                    "totalKeysExamined": 5000,
                    "totalDocsExamined": 5000,
                    "executionTimeMillis": 12
                }
            }
        }
    ]
}
```

A healthy pipeline has `nReturned` close to `totalDocsExamined`. A large gap indicates a missing index.

## Enabling the Database Profiler

Enable slow query profiling to capture aggregation pipelines that exceed a threshold:

```javascript
// Log all operations slower than 100ms
db.setProfilingLevel(1, { slowms: 100 });

// View recent slow operations
db.system.profile.find(
    { op: "command", "command.aggregate": { $exists: true } },
    { ns: 1, millis: 1, "command.pipeline": 1, ts: 1 }
).sort({ millis: -1 }).limit(20);
```

## Identifying Expensive Stages

Look for these warning signs in explain output and profiler entries:

```javascript
// Find pipelines using COLLSCAN (no index)
db.system.profile.find({
    "execStats.stage": "COLLSCAN",
    op: "command"
}).count();

// Find pipelines with high execution time
db.system.profile.find({
    millis: { $gt: 500 },
    "command.aggregate": { $exists: true }
}).sort({ millis: -1 });
```

## Checking allowDiskUse and Memory Usage

By default, aggregation pipelines have a 100MB memory limit per stage. When this is exceeded, MongoDB either throws an error or spills to disk if `allowDiskUse` is enabled:

```javascript
// Check if pipelines are spilling to disk
db.orders.explain("executionStats").aggregate(
    [
        { $sort: { amount: -1 } },
        { $group: { _id: "$category", max: { $max: "$amount" } } }
    ],
    { allowDiskUse: true }
);

// Look for usedDisk: true in explain output
```

## Using $planCacheStats for Aggregation Plans

```javascript
// View cached plans for aggregation queries
db.orders.aggregate([{ $planCacheStats: {} }]);
```

## Monitoring with mongotop and mongostat

```bash
# Watch collection-level read/write activity
mongotop --uri="mongodb://localhost:27017" 5

# Monitor server-wide operation counts
mongostat --uri="mongodb://localhost:27017" --humanReadable
```

## Atlas Performance Advisor

In MongoDB Atlas, the Performance Advisor automatically identifies slow aggregations and suggests indexes. Navigate to your cluster, then select "Performance Advisor" to see recommendations ranked by impact. Each recommendation shows the query pattern, the suggested index, and the estimated improvement.

## Creating Targeted Indexes for Pipelines

After identifying the slow stage, create an index that covers the `$match` filter and supports the sort:

```javascript
// For a pipeline starting with $match on status and createdAt, then sorting by customerId
db.orders.createIndex({
    status: 1,
    createdAt: 1,
    customerId: 1
});

// Verify the index is used
db.orders.explain("executionStats").aggregate([
    { $match: { status: "shipped", createdAt: { $gte: new Date("2024-01-01") } } },
    { $sort: { customerId: 1 } }
]);
```

## Summary

Monitoring aggregation pipeline performance requires using `explain("executionStats")` to understand stage-level execution, the database profiler to capture slow pipelines in production, and Atlas Performance Advisor for automated recommendations. The most impactful optimization is almost always adding an index that covers the `$match` stage at the beginning of the pipeline.
