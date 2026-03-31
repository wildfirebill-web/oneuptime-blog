# How to Fix MongoError: ExceededTimeLimit in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Time Limit, Timeout, Query Optimization, Error

Description: Learn why MongoDB throws ExceededTimeLimit errors and how to fix them by adding indexes, optimizing aggregations, and configuring appropriate timeout values.

---

## Understanding the Error

`MongoServerError: ExceededTimeLimit` (error code 262) means a MongoDB operation was killed because it consumed more CPU or I/O time than the server allowed. This is closely related to `MaxTimeMSExpired` (code 50) but can also occur during transactions or background operations:

```text
MongoServerError: ExceededTimeLimit: operation exceeded time limit
    code: 262, codeName: 'ExceededTimeLimit'
```

The error can also appear during transaction commit when the transaction takes longer than `transactionLifetimeLimitSeconds` (default: 60 seconds).

## Cause 1: Collection Scan on Large Collection

Without an index, MongoDB reads every document. On millions of records, this easily exceeds any reasonable timeout:

```javascript
// Check query plan
const result = await db.collection('events').find({
  userId: 'u123',
  type: 'purchase'
}).explain('executionStats');

console.log(result.queryPlanner.winningPlan); // look for COLLSCAN
console.log(result.executionStats.totalDocsExamined); // high number = problem
```

Add a compound index:

```javascript
await db.collection('events').createIndex({ userId: 1, type: 1, createdAt: -1 });
```

## Cause 2: Expensive Aggregation Without Early Filtering

```javascript
// BAD - $unwind before $match processes all documents
db.collection('orders').aggregate([
  { $unwind: "$items" },
  { $match: { "items.productId": "p123" } }
]);

// GOOD - $match first reduces documents early
db.collection('orders').aggregate([
  { $match: { "items.productId": "p123" } }, // filter before unwind
  { $unwind: "$items" },
  { $match: { "items.productId": "p123" } }  // re-filter after unwind
]);
```

## Cause 3: Transaction Taking Too Long

If a transaction runs longer than `transactionLifetimeLimitSeconds`, MongoDB aborts it with `ExceededTimeLimit`:

```javascript
// Reduce transaction scope - only include operations that must be atomic
const session = client.startSession();
await session.withTransaction(async () => {
  // Only the database operations here
  await db.collection('inventory').updateOne(
    { sku: 'ABC', qty: { $gte: 1 } },
    { $inc: { qty: -1 } },
    { session }
  );
  await db.collection('orders').insertOne(
    { sku: 'ABC', userId: 'u1', status: 'created' },
    { session }
  );
  // Do NOT do external API calls or heavy computation inside the session
});
session.endSession();
```

Increase the limit if transactions are legitimately long:

```javascript
db.adminCommand({ setParameter: 1, transactionLifetimeLimitSeconds: 120 })
```

## Cause 4: Large Sort Without Index

Sorting a large result set in memory may exceed time limits:

```javascript
// Add index to support the sort
await db.collection('logs').createIndex({ createdAt: -1 });

// Now this sort uses the index
await db.collection('logs').find({ level: 'error' })
  .sort({ createdAt: -1 })
  .limit(100)
  .toArray();
```

## Monitoring Long-Running Operations

Find operations that are running too long:

```javascript
db.adminCommand({ currentOp: true, secs_running: { $gt: 5 } })
```

Kill a specific operation:

```javascript
db.adminCommand({ killOp: 1, op: <opid> })
```

## Setting Per-Operation Timeout

Always set an explicit timeout for user-facing queries:

```javascript
const results = await db.collection('products').aggregate(pipeline, {
  maxTimeMS: 10000 // 10 seconds for user-facing queries
}).toArray();
```

## Summary

`ExceededTimeLimit` means a query, aggregation, or transaction took too long. Fix it by adding the right indexes, restructuring aggregation pipelines to filter early, keeping transactions short and focused on atomic database operations only, and setting appropriate `maxTimeMS` values for queries. Use `explain()` to identify collection scans and remove them.
