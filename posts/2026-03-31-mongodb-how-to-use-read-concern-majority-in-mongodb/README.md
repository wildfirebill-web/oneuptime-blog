# How to Use Read Concern 'majority' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Concern, Replication, Consistency, Durability, Transaction

Description: Learn how to use read concern majority in MongoDB to ensure reads only return data that has been durably committed to a majority of replica set members.

---

## Introduction

Read concern `majority` ensures that a read operation only returns data that has been acknowledged by a majority of replica set members. This guarantees that the data you read will not be rolled back even if the current primary fails. It is the recommended choice for applications that require strong consistency.

## How "majority" Works

When you issue a read with concern `majority`, MongoDB returns data from the majority-committed snapshot - a point in time where enough nodes have confirmed the write. This data is durable: even in a failover, it will persist.

The trade-off is slightly higher latency compared to `local`, because MongoDB must wait until replication has advanced to a majority-committed state.

## Setting Read Concern in mongosh

```javascript
db.orders.find({ status: "confirmed" }).readConcern("majority");
```

## Setting Read Concern in Node.js

```javascript
const { MongoClient, ReadConcern } = require("mongodb");
const client = new MongoClient("mongodb://localhost:27017");
await client.connect();

const collection = client.db("mydb").collection("orders");
const confirmed = await collection
  .find({ status: "confirmed" })
  .withReadConcern(new ReadConcern("majority"))
  .toArray();
```

## Using with Write Concern for Full Durability

Pair read concern `majority` with write concern `majority` for end-to-end durability:

```javascript
// Write with majority concern
await collection.insertOne(
  { orderId: "ORD-001", status: "confirmed" },
  { writeConcern: { w: "majority" } }
);

// Read back with majority concern
const order = await collection.findOne(
  { orderId: "ORD-001" },
  { readConcern: { level: "majority" } }
);
```

This pattern ensures the data you write is the data you read back.

## Behavior in Transactions

Using `majority` in a session transaction gives the highest consistency for multi-document operations:

```javascript
const session = client.startSession();
session.startTransaction({
  readConcern: { level: "majority" },
  writeConcern: { w: "majority" }
});
try {
  const account = await accounts.findOne({ userId }, { session });
  if (account.balance >= amount) {
    await accounts.updateOne({ userId }, { $inc: { balance: -amount } }, { session });
  }
  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
} finally {
  session.endSession();
}
```

## Enabling on the Replica Set

Read concern `majority` requires that `enableMajorityReadConcern` is enabled (it is the default in MongoDB 3.6+). Verify:

```javascript
db.adminCommand({ getParameter: 1, enableMajorityReadConcern: 1 });
```

## Summary

Read concern `majority` provides strong durability guarantees by only returning data confirmed by a majority of replica set members. Pairing it with write concern `majority` ensures your application operates on data that will survive a failover. Use it for financial transactions, order management, and any workflow where data loss is unacceptable.
