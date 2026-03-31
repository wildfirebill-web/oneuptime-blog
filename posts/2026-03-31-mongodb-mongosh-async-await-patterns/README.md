# How to Use mongosh with Async/Await Patterns

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongosh, Async, JavaScript

Description: Learn how to use async/await patterns in mongosh for parallel operations, error handling, retry logic, and async iteration over large cursor results.

---

## Top-Level Await in mongosh

mongosh supports top-level `await` because it wraps the REPL in an async context. You can use `await` directly without wrapping in an async function.

```javascript
// Top-level await works
const doc = await db.orders.findOne({ status: "pending" });
print(doc._id);

const count = await db.orders.countDocuments({ status: "active" });
print(`Active orders: ${count}`);
```

## Parallel Queries with Promise.all

Run multiple queries concurrently to reduce total latency:

```javascript
// Sequential - slow
const orders   = await db.orders.countDocuments();
const users    = await db.users.countDocuments();
const products = await db.products.countDocuments();

// Parallel - faster
const [orderCount, userCount, productCount] = await Promise.all([
  db.orders.countDocuments(),
  db.users.countDocuments(),
  db.products.countDocuments()
]);

print(`Orders: ${orderCount}, Users: ${userCount}, Products: ${productCount}`);
```

## Async Functions in mongosh

Define async functions for reusable logic:

```javascript
async function migrateDocuments(fromColl, toColl, filter) {
  const docs = await db[fromColl].find(filter).toArray();
  if (docs.length === 0) {
    print("No documents to migrate");
    return 0;
  }

  const session = db.getMongo().startSession();
  session.startTransaction();

  try {
    await db[toColl].insertMany(docs, { session });
    const ids = docs.map(d => d._id);
    await db[fromColl].deleteMany({ _id: { $in: ids } }, { session });
    await session.commitTransaction();
    print(`Migrated ${docs.length} documents`);
    return docs.length;
  } catch (err) {
    await session.abortTransaction();
    throw err;
  } finally {
    session.endSession();
  }
}

await migrateDocuments("orders", "orders_archive", { createdAt: { $lt: new Date("2024-01-01") } });
```

## Error Handling with Async/Await

```javascript
async function safeInsert(doc) {
  try {
    const result = await db.orders.insertOne(doc);
    return { ok: true, id: result.insertedId };
  } catch (err) {
    if (err.code === 11000) {
      return { ok: false, reason: "duplicate key" };
    }
    throw err;
  }
}

const r = await safeInsert({ _id: "ORD-001", amount: 100 });
printjson(r);
```

## Retry Logic with Async Loops

```javascript
async function withRetry(fn, maxAttempts = 3, delayMs = 500) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxAttempts) throw err;
      print(`Attempt ${attempt} failed: ${err.message}. Retrying...`);
      await new Promise(resolve => setTimeout(resolve, delayMs * attempt));
    }
  }
}

await withRetry(() => db.orders.insertOne({ amount: 50, ts: new Date() }));
```

## Processing Large Cursors with Async Iteration

```javascript
async function processBatch(batchSize = 100) {
  const cursor = db.events.find({ processed: false });
  let batch = [];
  let total = 0;

  for await (const doc of cursor) {
    batch.push(doc._id);

    if (batch.length >= batchSize) {
      await db.events.updateMany({ _id: { $in: batch } }, { $set: { processed: true } });
      total += batch.length;
      batch = [];
    }
  }

  if (batch.length > 0) {
    await db.events.updateMany({ _id: { $in: batch } }, { $set: { processed: true } });
    total += batch.length;
  }

  print(`Processed ${total} documents`);
}

await processBatch(200);
```

## Summary

mongosh's top-level `await` support makes async patterns natural in the shell. Use `Promise.all()` for parallel queries, async functions for reusable transaction logic, `try/catch` for error handling, and `for await...of` for async cursor iteration over large result sets. Retry patterns with `setTimeout` delays handle transient errors gracefully.
