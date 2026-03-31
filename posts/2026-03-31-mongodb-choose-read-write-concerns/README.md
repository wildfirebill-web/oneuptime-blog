# How to Choose Read and Write Concerns for Your Use Case in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Concern, Write Concern, Consistency, Architecture

Description: Learn how to choose the right combination of read and write concerns in MongoDB to balance consistency, durability, and performance for different application use cases.

---

## Why the Choice Matters

Read and write concerns in MongoDB control the trade-off between three properties:

- **Durability** - will the data survive a primary failure?
- **Consistency** - can a client immediately read what it just wrote?
- **Performance** - how much latency and throughput does this configuration cost?

No single setting is correct for all workloads. This guide maps use cases to the appropriate concern combinations.

## The Concern Matrix

```text
Write Concern   Read Concern     Durability   Consistency   Relative Cost
w:0             local            None         None          Lowest
w:1             local            Primary only Eventual      Low
w:1, j:true     local            Crash-safe   Eventual      Low-medium
w:majority      local            Yes          Eventual      Medium
w:majority      majority         Yes          Strong        Medium-high
w:majority      linearizable     Yes          Strict        Highest
```

## Use Case 1: High-Volume Event Ingestion

Analytics events, click tracking, and IoT sensor data can tolerate some data loss.

```javascript
db.events.insertMany(batch, {
  writeConcern: { w: 0 },
  ordered: false
});
```

**Recommended:** `w:0` for maximum throughput. No read concern needed as queries are approximate.

## Use Case 2: Application Logs

Logs should be written efficiently. Losing a handful of log entries on a crash is acceptable.

```javascript
db.logs.insertOne(logEntry, { writeConcern: { w: 1 } });
```

**Recommended:** `w:1` (default). Fast, primary-confirmed, but not crash-durable.

## Use Case 3: User Profile Updates

User data should not be lost, but some eventual consistency is acceptable.

```javascript
db.users.updateOne(
  { _id: userId },
  { $set: { profile: updatedProfile } },
  { writeConcern: { w: "majority", j: true } }
);
```

**Recommended:** `w:majority, j:true`. Durable and crash-safe on the primary. Read back with `readConcern: "local"` is fine since the primary just wrote the data.

## Use Case 4: Financial Transactions

Must not lose data and must be immediately consistent after write.

```javascript
const session = client.startSession();
session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority", j: true }
});

// Perform multi-document updates within transaction
await accounts.updateOne({ _id: from }, { $inc: { balance: -amount } }, { session });
await accounts.updateOne({ _id: to }, { $inc: { balance: amount } }, { session });

await session.commitTransaction();
```

**Recommended:** `w:majority, j:true` writes + `readConcern:"snapshot"` in transactions.

## Use Case 5: Read-Your-Writes Guarantee

Client writes data and immediately reads it back expecting to see it.

```javascript
// Write
await collection.insertOne(doc, { writeConcern: { w: "majority" } });

// Read - guaranteed to see the write above
const result = await collection.findOne(
  { _id: doc._id },
  { readConcern: { level: "majority" } }
);
```

**Recommended:** `w:majority` + `readConcern:"majority"` forms a causally consistent read-after-write.

## Use Case 6: Secondary Read Offloading (Analytics)

Offload heavy reporting queries to secondaries where some lag is acceptable.

```javascript
db.orders
  .find({ date: { $gte: startOfMonth } })
  .readPref("secondary")
  .readConcern("local");
```

**Recommended:** Read preference `secondary` + `readConcern:"local"`. Accepts replication lag in exchange for reduced primary load.

## Use Case 7: Distributed Locking

Requires the strongest guarantees to prevent split-brain scenarios.

```javascript
const lock = await db.locks.findOneAndUpdate(
  { _id: "resource_lock", held: false },
  { $set: { held: true, holder: clientId } },
  {
    writeConcern: { w: "majority" },
    readConcern: { level: "linearizable" },
    maxTimeMS: 5000
  }
);
```

**Recommended:** `w:majority` + `readConcern:"linearizable"` for the strongest consistency guarantee.

## Quick Reference Summary

```text
Use Case                 Write Concern        Read Concern
High-volume telemetry    w:0                  local
Application logging      w:1                  local
User data updates        w:majority, j:true   local
Financial writes         w:majority, j:true   snapshot (in txn)
Read-your-writes         w:majority           majority
Secondary analytics      w:majority           local (on secondary)
Distributed locking      w:majority           linearizable
```

## Summary

Choosing the right concerns requires understanding your application's tolerance for data loss and stale reads. Use `w:0` only for loss-tolerant metrics. Default to `w:1` for non-critical writes. Use `w:majority` plus `j:true` for business-critical data. Pair with `readConcern:"majority"` when you need to immediately read your own writes. Reserve `readConcern:"linearizable"` for strict coordination scenarios where latency is acceptable.
