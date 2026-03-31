# How to Use findAndModify() for Atomic Read-Modify-Write in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atomic, findAndModify, Operation, Driver

Description: Learn how to use MongoDB's findAndModify for atomic read-modify-write operations, and the modern findOneAndUpdate/findOneAndReplace alternatives.

---

`findAndModify` is MongoDB's command for performing an atomic find-and-update in a single round-trip, returning either the document before or after the modification. This eliminates race conditions inherent in separate read-then-write operations.

## Why findAndModify?

Without atomic find-and-modify, a read-then-write sequence is vulnerable to race conditions:

```javascript
// NOT ATOMIC - two clients can read the same value
const job = await db.collection("jobs").findOne({ status: "pending" });
await db.collection("jobs").updateOne(
  { _id: job._id },
  { $set: { status: "processing" } }
);
// Another worker may have already claimed this job
```

`findAndModify` atomically finds a document and updates it, returning the document in a single operation.

## Modern Alternatives: findOneAndUpdate/findOneAndReplace

The MongoDB drivers expose `findOneAndUpdate` and `findOneAndReplace` as the idiomatic way to use this functionality. The raw `findAndModify` command is a lower-level interface:

```javascript
// Find the next pending job and claim it atomically
const job = await db.collection("jobs").findOneAndUpdate(
  { status: "pending" },
  {
    $set: {
      status: "processing",
      startedAt: new Date(),
      workerId: workerId
    }
  },
  {
    sort: { priority: -1, createdAt: 1 },
    returnDocument: "after"
  }
);

if (job) {
  console.log(`Claimed job: ${job._id}`);
} else {
  console.log("No pending jobs available");
}
```

## Parameters Explained

- `filter` - the query to find the document
- `update` - the update operation or replacement document
- `sort` - if multiple documents match, determines which one to return
- `returnDocument: "before"` (default) - returns the document before modification
- `returnDocument: "after"` - returns the document after modification
- `upsert: true` - inserts if no document matches

## Sequence ID Generator Pattern

A classic use case - generating unique sequential IDs:

```javascript
async function getNextSequenceId(sequenceName) {
  const result = await db.collection("counters").findOneAndUpdate(
    { _id: sequenceName },
    { $inc: { seq: 1 } },
    {
      upsert: true,
      returnDocument: "after"
    }
  );
  return result.seq;
}

// Usage
const orderId = await getNextSequenceId("order_id");
console.log(`New order ID: ${orderId}`); // 1, 2, 3, ...
```

## Job Queue Pattern

```javascript
async function dequeueJob(workerId) {
  return db.collection("jobs").findOneAndUpdate(
    {
      status: "pending",
      scheduledFor: { $lte: new Date() }
    },
    {
      $set: {
        status: "processing",
        claimedBy: workerId,
        claimedAt: new Date()
      },
      $inc: { attempts: 1 }
    },
    {
      sort: { priority: -1, scheduledFor: 1 },
      returnDocument: "after"
    }
  );
}
```

## findOneAndDelete

Atomically find and remove a document:

```javascript
// Pop from a FIFO queue
const item = await db.collection("queue").findOneAndDelete(
  { status: "pending" },
  { sort: { createdAt: 1 } }
);
```

## Projection to Reduce Network Traffic

Return only needed fields:

```javascript
const job = await db.collection("jobs").findOneAndUpdate(
  { status: "pending" },
  { $set: { status: "processing" } },
  {
    projection: { _id: 1, payload: 1, type: 1 },
    returnDocument: "after"
  }
);
```

## Summary

Use `findOneAndUpdate` for atomic read-modify-write operations that need the document returned. It eliminates race conditions in patterns like job queue claiming, counter generation, and state machine transitions. The `sort` option determines which document is selected when multiple match. Use `returnDocument: "after"` to see the post-update state, or `"before"` to capture the previous state for logging or rollback purposes.
