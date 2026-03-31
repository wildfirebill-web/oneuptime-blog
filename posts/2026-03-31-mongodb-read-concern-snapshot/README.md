# How to Use Read Concern 'snapshot' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Concern, Transaction, Snapshot, Consistency

Description: Learn how MongoDB's read concern 'snapshot' provides a consistent point-in-time view across multiple documents within multi-document transactions.

---

## What Is Read Concern "snapshot"?

Read concern `"snapshot"` provides reads from a consistent snapshot of the data as of the transaction's start time. Within a multi-document transaction, every read operation sees the same state of the database - it reflects all writes that were committed before the transaction began, but no writes that happened concurrently during the transaction.

`"snapshot"` is the read concern designed specifically for multi-document transactions in MongoDB.

## How Snapshot Reads Work

When a transaction starts with `"snapshot"` read concern, MongoDB internally marks a "snapshot timestamp." All reads within the transaction return data as of that timestamp:

```text
T=0: Document A exists with balance=1000
T=1: Transaction starts (snapshot at T=0)
T=2: Another client updates balance=900 (not in transaction)
T=3: Transaction reads balance - returns 1000 (snapshot at T=0)
T=4: Transaction commits
```

The transaction never sees the concurrent update, preventing phantom reads.

## Setting Read Concern to "snapshot"

Read concern `"snapshot"` can only be used inside a multi-document transaction:

```javascript
const session = client.startSession();

session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
});

try {
  const sender = await accounts.findOne({ _id: "ACC-001" }, { session });
  const receiver = await accounts.findOne({ _id: "ACC-002" }, { session });

  // Both reads see the database as of the transaction start time
  await accounts.updateOne(
    { _id: "ACC-001" },
    { $inc: { balance: -100 } },
    { session }
  );
  await accounts.updateOne(
    { _id: "ACC-002" },
    { $inc: { balance: 100 } },
    { session }
  );

  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

## Why "snapshot" Prevents Anomalies

Without snapshot isolation, two reads in a transaction might return different states:

```text
Without snapshot isolation:
  T=1: Read account A balance = 1000
  T=2: Another client modifies account A to 900
  T=3: Read account A again = 900 (inconsistent within same transaction)

With "snapshot":
  T=1: Read account A balance = 1000 (snapshot locked)
  T=2: Another client modifies account A (not visible to transaction)
  T=3: Read account A again = 1000 (consistent snapshot)
```

## Usage Requirements

- Only available inside multi-document transactions
- Requires MongoDB 4.0+ for replica sets and 4.2+ for sharded clusters
- Cannot be used outside a transaction (returns an error)

## Sharded Cluster Support

On sharded clusters, `"snapshot"` read concern in transactions provides consistent reads across multiple shards:

```javascript
// Works across shards within a transaction
session.startTransaction({ readConcern: { level: "snapshot" } });

const result1 = await collection1.findOne({ _id: 1 }, { session }); // shard 1
const result2 = await collection2.findOne({ _id: 2 }, { session }); // shard 2
// Both see a consistent view as of the same snapshot timestamp
```

## Error Handling

If the snapshot is too old or a conflict occurs, MongoDB throws a `SnapshotTooOld` or `WriteConflict` error. Always implement retry logic:

```javascript
async function runWithRetry(fn, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const session = client.startSession();
    try {
      session.startTransaction({ readConcern: { level: "snapshot" } });
      await fn(session);
      await session.commitTransaction();
      return;
    } catch (err) {
      await session.abortTransaction();
      if (err.hasErrorLabel("TransientTransactionError") && attempt < maxRetries - 1) {
        continue;
      }
      throw err;
    } finally {
      session.endSession();
    }
  }
}
```

## Comparison With Other Concerns Inside Transactions

```text
Concern      Snapshot consistency   Rollback safe   Available outside txn
local        No                     No              Yes
majority     No (sees committed)    Yes             Yes
snapshot     Yes (point-in-time)    Yes             No
```

## Summary

Read concern `"snapshot"` provides a consistent, point-in-time view of the database for all reads within a multi-document transaction. It prevents dirty reads, non-repeatable reads, and phantom reads - the three classic transaction anomalies. Use it with `writeConcern: { w: "majority" }` for fully ACID-compliant transactions, and always implement retry logic for `TransientTransactionError` errors.
