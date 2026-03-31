# How to Fix MongoError: MaxTimeMSExpired in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Maxtime, Query Timeout, Performance, Error

Description: Learn why MongoDB throws MaxTimeMSExpired errors and how to fix them by optimizing queries, adding indexes, and tuning maxTimeMS limits.

---

## Understanding the Error

`MongoServerError: operation exceeded time limit` (error code 50, `MaxTimeMSExpired`) occurs when a query, aggregation, or command exceeds the time limit set by `maxTimeMS`. MongoDB's server kills the operation and returns this error.

```text
MongoServerError: operation exceeded time limit
    code: 50, codeName: 'MaxTimeMSExpired'
```

## Where maxTimeMS Comes From

The limit can be set explicitly per-operation or as a cluster-wide default:

```javascript
// Per-operation timeout
await db.collection('orders').find({ status: 'pending' })
  .maxTimeMS(5000) // 5 second limit
  .toArray();
```

```javascript
// Default server-side: set via Atlas or mongod parameter
// db.adminCommand({ setParameter: 1, defaultMaxTimeMS: 10000 })
```

## Cause 1: Missing or Unusable Index

The most common cause is a collection scan on a large collection. Add an index that covers the query:

```javascript
// Check the query plan
await db.collection('orders').find({ status: 'pending', userId: 'u123' })
  .explain('executionStats');
```

Look for `COLLSCAN` in the plan. Add a compound index:

```javascript
await db.collection('orders').createIndex({ userId: 1, status: 1 });
```

After adding the index, queries using both fields will use `IXSCAN` and execute much faster.

## Cause 2: Slow Aggregation Pipeline

Aggregation pipelines with `$lookup`, `$unwind`, or large `$group` stages can exceed time limits:

```javascript
// Profile the aggregation
await db.collection('orders').aggregate([
  { $match: { createdAt: { $gte: new Date('2025-01-01') } } },
  { $lookup: { from: 'products', localField: 'productId', foreignField: '_id', as: 'product' } },
  { $group: { _id: '$userId', total: { $sum: '$amount' } } }
], { explain: true });
```

Optimization strategies:
- Place `$match` first to reduce documents early
- Add indexes on `$lookup` foreign fields
- Use `$project` to remove unneeded fields before expensive stages

```javascript
// Optimized pipeline
await db.collection('orders').aggregate([
  { $match: { createdAt: { $gte: new Date('2025-01-01') }, status: 'completed' } },
  { $project: { userId: 1, amount: 1, productId: 1 } }, // reduce payload
  { $lookup: { from: 'products', localField: 'productId', foreignField: '_id', as: 'product' } },
  { $group: { _id: '$userId', total: { $sum: '$amount' } } }
]);
```

## Cause 3: maxTimeMS Set Too Low for the Workload

If the timeout is legitimate but the query is inherently slow (e.g., monthly report), increase the limit:

```javascript
// Long-running report with higher timeout
await db.collection('orders').aggregate(pipeline, { maxTimeMS: 120000 }); // 2 minutes
```

## Cause 4: Lock Contention

If the collection is under heavy write load, reads may wait for locks. Check current operations:

```javascript
db.currentOp({ "active": true, "secs_running": { $gt: 2 } })
```

Use read concern `"available"` or `"local"` for analytics workloads to reduce lock contention, or move reporting queries to a secondary.

## Increasing the Default Limit

For Atlas, set `defaultMaxTimeMS` at the cluster level (requires MongoDB 8.0+):

```javascript
db.adminCommand({ setParameter: 1, defaultMaxTimeMS: 30000 })
```

## Summary

`MaxTimeMSExpired` means a query took longer than allowed. Fix it by adding the right indexes, optimizing aggregation pipelines (filter early, project fields early), increasing `maxTimeMS` for legitimate long-running queries, and routing analytical queries to secondaries to avoid lock contention on the primary.
