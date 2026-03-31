# How to Handle MongoDB Lock Escalation Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Lock, Concurrency

Description: Understand MongoDB's locking model and fix lock contention issues caused by long-running operations, collection-level locks, and blocking writes.

---

## MongoDB's Locking Model

MongoDB uses intent locks at the global and database level and document-level locks for WiredTiger operations. Unlike older versions, WiredTiger does not use collection-level locks for most operations - it uses document-level concurrency control. However, certain operations still acquire collection or database-level exclusive locks:

```text
Global IX lock  - normal reads and writes
Global X lock   - initial sync, setFeatureCompatibilityVersion
Database IX/X   - collection creation, index builds
Collection IX/X - collection drops, chunk migration
Document lock   - individual document reads/writes
```

Long-running operations holding these higher-level locks block all other operations on the same resource.

## Detecting Lock Contention

```javascript
// Check the global lock queue
var gl = db.serverStatus().globalLock
print('Active readers:', gl.activeClients.readers)
print('Active writers:', gl.activeClients.writers)
print('Queued readers:', gl.currentQueue.readers)
print('Queued writers:', gl.currentQueue.writers)
```

High queue numbers indicate lock contention. Identify the blocking operation:

```javascript
db.currentOp({
  waitingForLock: true,
}).inprog.forEach(op => {
  print(`Waiting: OpId ${op.opid} | ${op.op} | ${op.ns} | ${op.secs_running}s`)
})

// Find the operation holding the lock
db.currentOp({
  waitingForLock: false,
  secs_running: { $gt: 5 },
}).inprog.forEach(op => {
  print(`Holding: OpId ${op.opid} | ${op.op} | ${op.ns} | ${op.secs_running}s`)
  printjson(op.locks)
})
```

## Killing the Blocking Operation

```javascript
// Kill a specific operation
db.adminCommand({ killOp: 1, op: 12345 })
```

## Index Builds and Lock Impact

Before MongoDB 4.2, creating indexes held an exclusive collection lock for the entire build duration. In MongoDB 4.2+, index builds are hybrid and only briefly hold an exclusive lock at the start and end:

```javascript
// Check active index builds
db.currentOp({
  $or: [
    { op: 'command', 'command.createIndexes': { $exists: true } },
    { 'command.commitIndexBuild': { $exists: true } },
  ],
})
```

Avoid running index builds during peak traffic. Schedule them during maintenance windows or use `maxTimeMS` to abort if they run too long:

```javascript
db.orders.createIndex({ userId: 1 }, { maxTimeMS: 3600000 })
```

## Avoiding Lock Contention in Application Code

**Problem:** Multiple application threads updating the same document simultaneously.

```javascript
// Contention-prone: all threads increment the same counter
await db.collection('counters').updateOne(
  { _id: 'pageviews' },
  { $inc: { count: 1 } }
)
```

**Solution:** Use per-shard or bucketed counters:

```javascript
// Distribute writes across 10 buckets
const bucket = Math.floor(Math.random() * 10)
await db.collection('counters').updateOne(
  { _id: `pageviews_${bucket}` },
  { $inc: { count: 1 } },
  { upsert: true }
)

// Read: sum all buckets
const buckets = await db.collection('counters').find({ _id: /^pageviews_/ }).toArray()
const total = buckets.reduce((sum, b) => sum + b.count, 0)
```

## Transactions and Lock Duration

Multi-document transactions hold locks until the transaction commits or aborts. Keep transactions short:

```javascript
const session = client.startSession()
session.startTransaction()
try {
  // Do minimal work inside transaction
  await orders.insertOne({ ... }, { session })
  await inventory.updateOne({ ... }, { $inc: { qty: -1 } }, { session })
  await session.commitTransaction()
} catch (err) {
  await session.abortTransaction()
  throw err
} finally {
  session.endSession()
}
```

Set `maxTransactionLockRequestTimeoutMillis` to prevent transactions from waiting indefinitely for a lock.

## Summary

MongoDB lock contention is diagnosed through `globalLock` metrics and `currentOp` with `waitingForLock: true`. The primary fixes are: killing long-running operations that hold locks, scheduling index builds during low-traffic periods, redesigning hot-document update patterns using bucketed counters, and keeping multi-document transactions as brief as possible.
