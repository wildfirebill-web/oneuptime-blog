# How to Use Write Concerns with Transactions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Transaction, Write Concern, Durability, Replica Set

Description: Understand MongoDB write concerns in transactions and how majority write concern ensures committed data survives replica set failovers.

---

## What Write Concern Does in Transactions

Write concern in MongoDB transactions controls when the commit acknowledgment is returned to the client. It determines how many replica set members must acknowledge the commit before the driver considers the transaction durable.

## Write Concern Options

The key write concern levels are:

- `w: 1` - acknowledged by the primary only; data could be lost on failover before replication
- `w: "majority"` - acknowledged by a majority of replica set members; survives single-node failure
- `j: true` - requires the write to be written to the journal before acknowledgment

## Setting Write Concern on a Transaction

Always set write concern at `startTransaction`, not on individual operations within the transaction.

```javascript
const session = client.startSession();
session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority", j: true }
});
```

Setting it on individual operations inside a transaction is ignored - the transaction-level write concern applies to the commit.

## Why majority Write Concern Matters

Without `w: "majority"`, a committed transaction could be rolled back after a primary failover if the secondary elected as new primary did not yet replicate the committed data.

```javascript
// Unsafe - w:1 allows data loss after primary failover
session.startTransaction({
  writeConcern: { w: 1 }
});

// Safe - majority ensures durability
session.startTransaction({
  writeConcern: { w: "majority" }
});
```

## Full Transaction Example with Write Concern

```javascript
async function transferFunds(client, fromId, toId, amount) {
  const session = client.startSession();
  try {
    await session.withTransaction(
      async () => {
        const accounts = client.db("bank").collection("accounts");

        const from = await accounts.findOne({ _id: fromId }, { session });
        if (from.balance < amount) {
          throw new Error("Insufficient funds");
        }

        await accounts.updateOne(
          { _id: fromId },
          { $inc: { balance: -amount } },
          { session }
        );

        await accounts.updateOne(
          { _id: toId },
          { $inc: { balance: amount } },
          { session }
        );
      },
      {
        readConcern: { level: "snapshot" },
        writeConcern: { w: "majority", j: true }
      }
    );
    console.log("Transfer completed");
  } finally {
    await session.endSession();
  }
}
```

## Write Concern and Transaction Latency

`w: "majority"` adds latency proportional to replication lag. To measure the impact:

```javascript
const start = Date.now();
await session.commitTransaction();
console.log(`Commit latency: ${Date.now() - start}ms`);
```

If commit latency is too high, ensure secondaries are on the same network segment or datacenter as the primary.

## Summary

MongoDB transaction write concern should always be set to `w: "majority"` for production use to guarantee that committed transactions survive replica set failovers. Set write concern on `startTransaction` or `withTransaction` options rather than on individual operations. Adding `j: true` further ensures data is journaled before acknowledgment, providing the strongest durability guarantee.
