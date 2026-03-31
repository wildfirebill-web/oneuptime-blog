# How to Use Causal Consistency in MongoDB Sessions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Causal Consistency, Session, Replica Set, Read Your Writes

Description: Learn how to use MongoDB causally consistent sessions to guarantee that reads always reflect your own prior writes in a replica set.

---

## What Is Causal Consistency?

In a MongoDB replica set, reads are typically routed to secondaries for scalability. This can cause a read-your-own-writes problem: you write to the primary, then immediately read from a secondary that has not yet replicated the change.

Causal consistency ensures that within a session, operations are executed in a logical causal order:
- Read your own writes
- Monotonic reads (reads never go backward in time)
- Monotonic writes
- Writes follow reads

## Creating a Causally Consistent Session

```javascript
const { MongoClient } = require("mongodb")

const client = new MongoClient("mongodb+srv://cluster.mongodb.net/mydb")
await client.connect()

// Open a causally consistent session
const session = client.startSession({
  causalConsistency: true  // default is true for sessions
})
```

## Read Your Own Writes Example

Without causal consistency:

```javascript
// Without causal consistency - may not see the write
await db.collection("profiles").updateOne(
  { userId: "user123" },
  { $set: { displayName: "Alice Smith" } }
)

// If reading from a secondary, this might return old data
const profile = await db.collection("profiles").findOne(
  { userId: "user123" },
  { readPreference: "secondaryPreferred" }
)
console.log(profile.displayName)  // might still be "Alice"
```

With causal consistency:

```javascript
const session = client.startSession({ causalConsistency: true })

try {
  // Write using the session
  await db.collection("profiles").updateOne(
    { userId: "user123" },
    { $set: { displayName: "Alice Smith" } },
    { session }
  )

  // This read is guaranteed to see the preceding write
  const profile = await db.collection("profiles").findOne(
    { userId: "user123" },
    {
      session,
      readPreference: "secondaryPreferred"  // still routes to secondary if possible
    }
  )
  console.log(profile.displayName)  // guaranteed: "Alice Smith"
} finally {
  await session.endSession()
}
```

## How It Works Internally

MongoDB tracks causality using `operationTime` and `clusterTime` tokens:

```javascript
const session = client.startSession({ causalConsistency: true })

// After a write, the session records the operation time
await db.collection("orders").insertOne(
  { orderId: "ORD-001", status: "pending" },
  { session }
)

console.log(session.operationTime)  // Timestamp indicating when this op occurred

// Subsequent reads include afterClusterTime to ensure the replica
// has caught up to at least this point
const order = await db.collection("orders").findOne(
  { orderId: "ORD-001" },
  { session }
)
```

## Combining with Read Preferences

Causal consistency works with all read preferences:

```javascript
const session = client.startSession({ causalConsistency: true })

// Write to primary
await db.collection("inventory").updateOne(
  { item: "widget" },
  { $inc: { qty: -1 } },
  { session, writeConcern: { w: "majority" } }
)

// Read from secondary - causal consistency ensures we see our write
const stock = await db.collection("inventory").findOne(
  { item: "widget" },
  {
    session,
    readPreference: "nearest",  // can use any preference
    readConcern: { level: "majority" }  // recommended with causal consistency
  }
)
```

## Performance Considerations

Causal consistency adds some overhead because secondaries must wait until they have replicated up to the `afterClusterTime` before responding:

```javascript
// For maximum performance, avoid causal consistency when not needed
// Only use it when read-your-own-writes semantics are required

// Non-causally-consistent session (slightly faster)
const fastSession = client.startSession({ causalConsistency: false })

// Causally consistent (adds secondary wait time)
const consistentSession = client.startSession({ causalConsistency: true })
```

## Session Scope and Propagation

Sessions are not shared across requests by default. To propagate causal consistency across service calls, pass the operation time:

```javascript
// Service A: performs write and returns its operationTime
async function updateProfile(userId, data) {
  const session = client.startSession({ causalConsistency: true })
  await db.collection("profiles").updateOne(
    { userId },
    { $set: data },
    { session, writeConcern: { w: "majority" } }
  )
  const opTime = session.operationTime
  await session.endSession()
  return { success: true, operationTime: opTime }
}

// Service B: advances its session to respect the previous operation
async function readProfile(userId, afterTime) {
  const session = client.startSession({ causalConsistency: true })
  if (afterTime) {
    session.advanceOperationTime(afterTime)
  }
  const profile = await db.collection("profiles").findOne({ userId }, { session })
  await session.endSession()
  return profile
}
```

## Requirements for Causal Consistency

```text
- MongoDB 3.6 or later
- Replica set or sharded cluster (not standalone)
- readConcern "majority" and writeConcern "majority" recommended
- Driver support: Node.js, Python, Java, Go, and others support sessions
```

## Summary

Causally consistent sessions in MongoDB guarantee that a client reads its own writes and sees a monotonically advancing view of the data, even when reads are routed to secondaries. Create a session with `causalConsistency: true`, pass the session object to all read and write operations within a logical unit of work, and close the session when done. Use majority read and write concerns alongside causal consistency for the strongest guarantees.
