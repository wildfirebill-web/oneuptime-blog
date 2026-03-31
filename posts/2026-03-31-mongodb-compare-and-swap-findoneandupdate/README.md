# How to Implement Compare-and-Swap with findOneAndUpdate in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Concurrency, Atomic Operation, findOneAndUpdate, Optimistic Locking

Description: Learn how to implement compare-and-swap (CAS) semantics in MongoDB using findOneAndUpdate to safely manage concurrent state transitions.

---

## What Is Compare-and-Swap?

Compare-and-swap (CAS) is a concurrency primitive: update a value only if it currently matches an expected value. If the value has changed since you last read it, the operation fails and you retry. This avoids the lost-update problem without pessimistic locks.

MongoDB does not have a dedicated CAS instruction, but `findOneAndUpdate` with a conditional filter achieves exactly the same guarantee.

## Basic CAS Pattern

Suppose you store a version counter on each document. Before writing, you assert that the version matches what you read:

```javascript
const doc = await Item.findById(itemId).lean();
const expectedVersion = doc.version;

const result = await Item.findOneAndUpdate(
  { _id: itemId, version: expectedVersion },
  {
    $set: { status: "processed", updatedAt: new Date() },
    $inc: { version: 1 }
  },
  { returnDocument: "after" }
);

if (!result) {
  // Another process changed the document - retry or surface conflict
  throw new Error("Version conflict: document was modified concurrently");
}
```

The filter `{ _id: itemId, version: expectedVersion }` is the "compare" step. The `$inc: { version: 1 }` is the "swap". If any concurrent writer incremented the version first, the filter matches nothing and `result` is `null`.

## CAS for State Machine Transitions

CAS shines for state machine workflows where a document must be in a specific state before transitioning:

```javascript
async function approveOrder(orderId) {
  const result = await Order.findOneAndUpdate(
    { _id: orderId, status: "pending" },
    { $set: { status: "approved", approvedAt: new Date() } },
    { returnDocument: "after" }
  );

  if (!result) {
    const current = await Order.findById(orderId).select("status");
    throw new Error(`Cannot approve order in state: ${current?.status}`);
  }
  return result;
}
```

Even if two workers try to approve the same order simultaneously, only one will match the `status: "pending"` filter. The second gets `null` and handles the conflict gracefully.

## Using returnDocument for Post-Update Logic

`returnDocument: "after"` returns the document as it looks after the update. Use `"before"` when you need the original state for logging or rollback:

```javascript
const before = await Job.findOneAndUpdate(
  { _id: jobId, lockedBy: null },
  { $set: { lockedBy: workerId, lockedAt: new Date() } },
  { returnDocument: "before" }
);

if (!before) {
  console.log("Job already claimed by another worker");
  return;
}
// before.lockedBy is still null here - useful for audit logging
```

## Retry Logic with Exponential Backoff

Under high contention, CAS failures are expected. Implement retries:

```javascript
async function casWithRetry(id, expectedVersion, updates, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const result = await Item.findOneAndUpdate(
      { _id: id, version: expectedVersion },
      { ...updates, $inc: { version: 1 } },
      { returnDocument: "after" }
    );

    if (result) return result;

    // Re-read and try again
    const fresh = await Item.findById(id).lean();
    if (!fresh) throw new Error("Document not found");
    expectedVersion = fresh.version;
    Object.assign(updates.$set || {}, computeUpdates(fresh));

    const delay = Math.min(100 * Math.pow(2, attempt), 1000);
    await new Promise(res => setTimeout(res, delay));
  }
  throw new Error("Max retries exceeded: too much contention");
}
```

## CAS with Timestamps Instead of Versions

If adding a version field is not practical, use a timestamp:

```javascript
const result = await Config.findOneAndUpdate(
  { _id: "app-config", updatedAt: lastReadAt },
  { $set: { value: newValue, updatedAt: new Date() } },
  { returnDocument: "after" }
);
```

Timestamps are less reliable than incrementing integers under very high throughput because two writes within the same millisecond could both pass. Prefer an explicit version counter for critical state.

## Summary

MongoDB's `findOneAndUpdate` provides CAS semantics through its conditional filter parameter. Include the expected current value - a version number, status field, or timestamp - in the query filter. If the assertion fails the operation returns `null` and you retry with fresh data. This pattern eliminates read-modify-write races for single-document state transitions without the overhead of multi-document transactions.
