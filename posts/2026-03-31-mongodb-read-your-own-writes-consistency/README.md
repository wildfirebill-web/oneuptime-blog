# How to Use Read-Your-Own-Writes Consistency in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Consistency, Session, Replica Set, Causal Consistency

Description: Learn how to ensure read-your-own-writes consistency in MongoDB using causally consistent sessions, so reads always reflect your application's most recent writes.

---

## The Read-Your-Own-Writes Problem

In a MongoDB replica set, reads from secondaries can lag behind the primary by seconds or even longer. If your application writes to the primary and then immediately reads from a secondary, the secondary may not yet have replicated the write. The user sees stale data - as if their action had no effect.

MongoDB solves this with read-your-own-writes (RYOW) consistency, which guarantees that a session will always see its own prior writes, regardless of which node serves the read.

## How Causal Consistency Enables RYOW

MongoDB implements RYOW through causally consistent sessions. A session tracks an operation time (cluster time) and a session token. When a read is issued in the session, MongoDB ensures the serving node has applied all writes that occurred before the read in the same session.

```javascript
const { MongoClient } = require("mongodb");

async function updateAndVerify(userId, newEmail) {
  const client = new MongoClient("mongodb://localhost:27017", {
    readPreference: "secondaryPreferred"  // Prefer secondaries for reads
  });
  await client.connect();

  // Start a causally consistent session
  const session = client.startSession({ causalConsistency: true });

  const users = client.db("app").collection("users");

  try {
    // Write to primary
    await users.updateOne(
      { _id: userId },
      { $set: { email: newEmail } },
      { session }
    );

    // Read from secondary - guaranteed to see the above write
    const user = await users.findOne(
      { _id: userId },
      { session, readPreference: "secondaryPreferred" }
    );

    console.log(`Email updated to: ${user.email}`);  // Will show newEmail
  } finally {
    await session.endSession();
    await client.close();
  }
}
```

## Python Example

```python
import pymongo

client = pymongo.MongoClient("mongodb://localhost:27017")

with client.start_session(causal_consistency=True) as session:
    users = client["app"]["users"]

    # Write
    users.update_one(
        {"_id": "user-123"},
        {"$set": {"plan": "premium"}},
        session=session
    )

    # Read - guaranteed to see the above write
    user = users.find_one(
        {"_id": "user-123"},
        session=session
    )

    print(f"Plan: {user['plan']}")  # Prints: Plan: premium
```

## Using afterClusterTime for Cross-Session RYOW

For RYOW across different sessions (e.g., in microservices), pass the operation time from the write session to the read session:

```javascript
// In the write service
const writeSession = client.startSession({ causalConsistency: true });
await collection.insertOne({ data: "important" }, { session: writeSession });

// Get the cluster time of the write
const clusterTime = writeSession.clusterTime;
const operationTime = writeSession.operationTime;
await writeSession.endSession();

// In the read service - pass cluster time to ensure RYOW
const readSession = client.startSession({ causalConsistency: true });
readSession.advanceClusterTime(clusterTime);
readSession.advanceOperationTime(operationTime);

const doc = await collection.findOne({ data: "important" }, { session: readSession });
await readSession.endSession();
```

## Read Concern for RYOW

Combine causally consistent sessions with `majority` read concern for the strongest RYOW guarantee that survives failover:

```javascript
const result = await collection.findOne(
  { _id: docId },
  {
    session,
    readConcern: { level: "majority" }
  }
);
```

## Summary

Read-your-own-writes consistency in MongoDB is achieved through causally consistent sessions. When reads and writes share a session with `causalConsistency: true`, MongoDB tracks operation times and ensures each read reflects all prior writes within that session. This is essential when using `secondaryPreferred` read preference and wanting to avoid showing users stale data after their own writes. For cross-service RYOW, pass operation time tokens between services to advance the session's causal clock.
