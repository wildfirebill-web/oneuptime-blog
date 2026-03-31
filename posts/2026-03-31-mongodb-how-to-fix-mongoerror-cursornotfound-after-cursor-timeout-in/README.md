# How to Fix MongoError: CursorNotFound After Cursor Timeout in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, CursorNotFound, Cursor Timeout, Troubleshooting, Query Optimization

Description: Learn how to fix the MongoError CursorNotFound error caused by server-side cursor expiration during long-running batch operations or slow iteration.

---

## Overview

MongoDB server-side cursors expire after 10 minutes of inactivity by default. If your application iterates slowly through a large result set and does not request the next batch within that window, the server kills the cursor and returns:

```text
MongoError: cursor id 1234567890 not found
MongoServerError: cursor id 1234567890 not found
```

## Causes

1. Iterating through a cursor too slowly (processing each document takes too long).
2. Network pause or application stall between batch fetches.
3. Using cursors inside a loop with slow I/O (e.g., writing to a slow API for each document).
4. Leaving a cursor open while performing other long-running operations.

## Fix 1 - Fetch All Results with toArray()

If your result set fits in memory, fetch everything upfront and process offline:

```javascript
// PROBLEM: cursor times out during slow processing
const cursor = db.collection("orders").find({ status: "pending" });

for await (const doc of cursor) {
  await slowApiCall(doc);  // each call takes 2s, 10k docs = timeout
}

// FIX: fetch all first, then process
const orders = await db.collection("orders")
  .find({ status: "pending" })
  .toArray();

for (const order of orders) {
  await slowApiCall(order);
}
```

## Fix 2 - Use NoCursorTimeout Option

Set `noCursorTimeout: true` to prevent the server from killing the cursor. Use with caution - always close the cursor when done:

```javascript
const cursor = db.collection("orders").find(
  { status: "pending" },
  { noCursorTimeout: true }
);

try {
  for await (const doc of cursor) {
    await processDocument(doc);
  }
} finally {
  await cursor.close();  // IMPORTANT: always close to free server resources
}
```

## Fix 3 - Process in Batches to Stay Active

Fetch in controlled batches to keep the cursor alive through activity:

```javascript
async function processBatched(collection, filter, batchSize = 200) {
  let skip = 0;
  let hasMore = true;

  while (hasMore) {
    const batch = await collection
      .find(filter)
      .skip(skip)
      .limit(batchSize)
      .toArray();

    if (batch.length === 0) {
      hasMore = false;
      break;
    }

    await Promise.all(batch.map(processDocument));
    skip += batchSize;
    console.log(`Processed ${skip} documents`);
  }
}
```

## Fix 4 - Use Range-Based Pagination Instead of Skip

For large collections, skip-based pagination is slow. Use `_id`-based range pagination:

```javascript
async function processAllOrders(collection) {
  let lastId = null;
  const batchSize = 500;

  while (true) {
    const filter = lastId
      ? { status: "pending", _id: { $gt: lastId } }
      : { status: "pending" };

    const batch = await collection
      .find(filter)
      .sort({ _id: 1 })
      .limit(batchSize)
      .toArray();

    if (batch.length === 0) break;

    for (const doc of batch) {
      await processDocument(doc);
    }

    lastId = batch[batch.length - 1]._id;
    console.log("Last processed ID:", lastId);
  }
}
```

## Fix 5 - Increase Cursor Timeout (Server-Side)

Increase the default cursor timeout on the server (in milliseconds):

```javascript
db.adminCommand({
  setParameter: 1,
  cursorTimeoutMillis: 1800000  // 30 minutes
});
```

In `mongod.conf`:

```yaml
setParameter:
  cursorTimeoutMillis: 1800000
```

This is a server-wide change - prefer application-level fixes when possible.

## Fix 6 - Use Aggregation with $out for Bulk Transformations

For data transformation jobs, use aggregation with `$out` to avoid long-running cursors entirely:

```javascript
// Process and write results in one server-side operation
await db.collection("orders").aggregate([
  { $match: { status: "pending" } },
  { $addFields: { processedAt: new Date() } },
  { $out: "processedOrders" }
]).toArray();
```

## Handling the Error Gracefully

If you cannot prevent the timeout, implement retry logic:

```javascript
async function safeIterate(collection, filter, processor) {
  let lastId = null;
  let retries = 0;

  while (true) {
    const query = lastId ? { ...filter, _id: { $gt: lastId } } : filter;

    try {
      const batch = await collection
        .find(query)
        .sort({ _id: 1 })
        .limit(100)
        .toArray();

      if (batch.length === 0) break;

      for (const doc of batch) {
        await processor(doc);
        lastId = doc._id;
      }

      retries = 0;
    } catch (err) {
      if (err.code === 43 && retries < 3) {  // CursorNotFound error code
        console.warn("Cursor lost, resuming from last processed ID:", lastId);
        retries++;
        continue;
      }
      throw err;
    }
  }
}
```

## Summary

The `CursorNotFound` error occurs when the server-side cursor expires due to inactivity during slow iteration. The cleanest solutions are: using `toArray()` for manageable result sets, enabling `noCursorTimeout` (with mandatory cursor close), or restructuring iteration into fixed-size batches with range-based pagination. Avoid relying on a single long-lived cursor for bulk processing jobs - use batched queries or server-side aggregation with `$out` instead.
