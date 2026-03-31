# How to Monitor and Reduce Lock Contention in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Performance, Lock, Concurrency, Monitoring

Description: Learn how to detect lock contention in MongoDB using serverStatus and currentOp, and apply practical strategies to reduce it and improve throughput.

---

## What Is Lock Contention in MongoDB?

MongoDB uses a multi-granularity locking system. When multiple operations compete to acquire locks at the same level (database or collection), some operations must wait. This waiting is lock contention, and it degrades throughput and increases latency.

High lock contention is often a symptom of:
- Long-running write operations blocking readers.
- Unindexed queries holding collection-level locks during scan.
- Write-heavy workloads without batching.

## Detecting Lock Contention with serverStatus

The `serverStatus` command exposes global lock metrics:

```javascript
db.serverStatus().globalLock;
```

Key fields to examine:

```json
{
  "globalLock": {
    "currentQueue": {
      "total": 12,
      "readers": 5,
      "writers": 7
    },
    "activeClients": {
      "total": 20,
      "readers": 10,
      "writers": 10
    }
  }
}
```

If `currentQueue.total` is consistently above zero, operations are queuing for locks. A growing writer queue is a strong signal of contention.

## Viewing Active Operations with currentOp

Use `currentOp` to see what is currently running and how long it has been waiting:

```javascript
db.adminCommand({
  currentOp: true,
  waitingForLock: true
});
```

This returns only operations currently blocked waiting for a lock. Look at the `secs_running` and `lockStats` fields:

```json
{
  "op": "update",
  "ns": "app.orders",
  "secs_running": 45,
  "waitingForLock": true,
  "lockStats": {
    "Collection": {
      "acquireWaitCount": { "W": 1 },
      "timeAcquiringMicros": { "W": 45000000 }
    }
  }
}
```

An operation waiting 45 seconds for a write lock on a collection is a serious problem.

## Killing a Long-Running Operation

If a runaway operation is causing a queue, kill it by its `opid`:

```javascript
db.adminCommand({ killOp: 1, op: 12345 });
```

## Strategies to Reduce Lock Contention

### Add Indexes for Write Predicates

Unindexed updates scan the collection and hold the lock longer. Add indexes on the fields used in `$match` or update filter conditions:

```javascript
db.orders.createIndex({ status: 1, updatedAt: 1 });
```

### Use Smaller, Targeted Writes

Instead of one large update that locks the collection for a long time, break it into smaller batches:

```javascript
// Process 500 documents at a time instead of all at once
const ids = db.orders.find({ status: "old" }, { _id: 1 }).limit(500).toArray().map(d => d._id);
db.orders.updateMany(
  { _id: { $in: ids } },
  { $set: { status: "archived" } }
);
```

### Use WiredTiger's Document-Level Concurrency

MongoDB's WiredTiger storage engine provides document-level concurrency control. Ensure you are not forcing collection-level locks by using multi-document transactions only when truly needed.

### Separate Read and Write Workloads

Use read replicas with appropriate `readPreference` settings to offload analytical reads from the primary, which handles writes:

```javascript
const client = new MongoClient(uri, {
  readPreference: "secondaryPreferred"
});
```

## Summary

Lock contention in MongoDB shows up as queued operations in `serverStatus().globalLock.currentQueue`. Use `currentOp` to find the blocking operations and kill runaway jobs quickly. Long-term reduction requires adding indexes on write predicates, batching large writes, and routing read traffic to secondary replicas.
