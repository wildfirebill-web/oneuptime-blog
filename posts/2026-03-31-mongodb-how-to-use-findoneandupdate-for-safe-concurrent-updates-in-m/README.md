# How to Use findOneAndUpdate for Safe Concurrent Updates in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, findOneAndUpdate, Concurrency, Atomic, Update

Description: Learn how to use MongoDB's findOneAndUpdate for atomic read-modify-write operations that prevent race conditions in concurrent applications.

---

## Why findOneAndUpdate

Regular read-then-update patterns have a race condition: between reading and writing, another process can modify the document. `findOneAndUpdate` atomically finds a matching document and applies an update in a single operation.

## Basic Syntax

```javascript
const result = await db.collection("inventory").findOneAndUpdate(
  { sku: "widget-42", quantity: { $gt: 0 } },
  { $inc: { quantity: -1 } },
  {
    returnDocument: "after",  // return the updated document
    upsert: false
  }
);

if (!result) {
  console.log("Item out of stock");
}
```

## Options Reference

Key options for `findOneAndUpdate`:

```javascript
{
  returnDocument: "before" | "after",  // which version to return
  upsert: true | false,                // insert if no match
  sort: { priority: -1 },             // pick which document to update when multiple match
  projection: { name: 1, status: 1 }, // return only specific fields
  maxTimeMS: 5000                     // timeout in milliseconds
}
```

## Job Queue Pattern

Atomically claim a job so only one worker processes it:

```javascript
async function claimNextJob(workerId) {
  return db.collection("jobs").findOneAndUpdate(
    {
      status: "queued",
      scheduledAt: { $lte: new Date() }
    },
    {
      $set: {
        status: "running",
        workerId: workerId,
        startedAt: new Date()
      }
    },
    {
      sort: { priority: -1, scheduledAt: 1 },
      returnDocument: "after"
    }
  );
}
```

## Seat Reservation Pattern

Reserve the last available seat without overbooking:

```javascript
async function reserveSeat(eventId, userId) {
  const result = await db.collection("events").findOneAndUpdate(
    {
      _id: eventId,
      availableSeats: { $gt: 0 }
    },
    {
      $inc: { availableSeats: -1 },
      $push: {
        reservations: {
          userId: userId,
          reservedAt: new Date()
        }
      }
    },
    { returnDocument: "after" }
  );

  if (!result) throw new Error("No seats available");
  return result;
}
```

## Upsert with findOneAndUpdate

Create a document if it does not exist, or update it if it does:

```javascript
const counter = await db.collection("counters").findOneAndUpdate(
  { _id: "pageviews" },
  { $inc: { count: 1 } },
  {
    upsert: true,
    returnDocument: "after"
  }
);

console.log("Total views:", counter.count);
```

## Handling the null Return

When no document matches the filter, `findOneAndUpdate` returns `null` (not an error):

```javascript
const updated = await collection.findOneAndUpdate(filter, update, options);

if (updated === null) {
  // Handle: document not found, condition not met, or race condition lost
  throw new AppError("UPDATE_FAILED", "Document not found or condition not met");
}
```

## Performance Tips

- Always include indexed fields in the filter for fast lookups
- Use `sort` carefully - sorting large result sets before finding the match is expensive
- Prefer specific filters that match one document to avoid unnecessary scanning

## Summary

`findOneAndUpdate` provides atomic find-and-modify semantics, making it ideal for job queues, counters, reservations, and any read-modify-write operation. The key options are `returnDocument` (choose before or after state), `upsert` for create-or-update, and `sort` for picking among multiple matches.
