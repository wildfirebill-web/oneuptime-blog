# How to Use Read Concern 'majority' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Concern, Consistency, Replica Set, Durability

Description: Learn how MongoDB's read concern 'majority' guarantees durable reads that cannot be rolled back, and when to use it for strong consistency requirements.

---

## What Is Read Concern "majority"?

Read concern `"majority"` returns only data that has been acknowledged and written by a majority of the replica set members. Data returned by a `"majority"` read is guaranteed to be durable - it cannot be rolled back even if the current primary fails and a new primary is elected.

This is the recommended read concern for applications that require strong consistency guarantees.

## How It Works

MongoDB maintains a "majority commit point" - the highest oplog timestamp that a majority of replica set members have applied. A `"majority"` read returns data as of that commit point, not the absolute latest data on the node.

```text
Timeline:
  Primary writes op at T=100
  Secondary 1 applies op at T=101
  Secondary 2 applies op at T=102
  Majority commit point advances to T=100
  readConcern:"majority" now includes the op
```

## Setting Read Concern to "majority"

At the query level:

```javascript
db.orders.find({ status: "shipped" }).readConcern("majority")
```

At the session level:

```javascript
const session = client.startSession({
  defaultTransactionOptions: {
    readConcern: { level: "majority" }
  }
});
```

At the driver connection level (Node.js):

```javascript
const client = new MongoClient(uri, {
  readConcernLevel: "majority"
});
```

## Comparison With "local"

```text
Concern      Returns                 Rollback safe   Latency
local        Node's latest data      No              Lowest
majority     Majority-committed      Yes             Moderate
```

`"local"` reads are faster but carry rollback risk. `"majority"` reads add a small latency cost to wait for the majority commit point.

## When to Use "majority"

Use read concern `"majority"` when:

- Reading data immediately after a `w:majority` write to confirm durability
- Building financial or inventory systems where stale or rolled-back reads are unacceptable
- Reading data inside transactions where you require consistent, durable snapshots
- Implementing causal consistency with `afterClusterTime` sessions

## Causal Consistency Example

Combine `w:majority` write with `"majority"` read to ensure a client always reads its own writes:

```javascript
// Write with majority durability
await collection.insertOne(
  { orderId: "ORD-001", status: "confirmed" },
  { writeConcern: { w: "majority" } }
);

// Read back with majority - guaranteed to see the write
const order = await collection.findOne(
  { orderId: "ORD-001" },
  { readConcern: { level: "majority" } }
);
```

## Use in Transactions

Within multi-document transactions, `"majority"` is the recommended read concern:

```javascript
const session = client.startSession();
session.startTransaction({
  readConcern: { level: "majority" },
  writeConcern: { w: "majority" }
});

try {
  const account = await accounts.findOne({ _id: "ACC-1" }, { session });
  await accounts.updateOne(
    { _id: "ACC-1" },
    { $inc: { balance: -100 } },
    { session }
  );
  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
}
session.endSession();
```

## Requirements

- Requires a replica set or sharded cluster (not available on standalone)
- The storage engine must support majority read concern (WiredTiger does; MMAPv1 did not)
- Has no additional configuration required beyond the replica set

## Performance Impact

`"majority"` reads wait for the majority commit point before returning data. In a healthy replica set with low replication lag, this is milliseconds. In a degraded replica set where secondaries are lagging significantly, reads may block or return older data to respect the commit point.

## Summary

Read concern `"majority"` ensures that returned data has been durably committed to a majority of replica set members and cannot be rolled back. Use it when data correctness matters more than the lowest possible read latency. Pair it with `w:majority` writes for fully consistent read-your-writes semantics, especially in transaction-heavy applications.
