# How to Use initializeUnorderedBulkOp() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, BulkWrite, Performance, Database, Node.js

Description: Learn how to use initializeUnorderedBulkOp() in MongoDB to execute bulk operations in parallel without order dependency, maximizing write throughput.

---

## What Is initializeUnorderedBulkOp()

`initializeUnorderedBulkOp()` creates a bulk write builder where operations are executed independently and in parallel. Unlike the ordered variant, if one operation fails, the remaining operations continue executing. The results are returned in an arbitrary order.

This approach is best when your operations are independent of each other and you prioritize throughput over sequential consistency.

## Basic Usage

```javascript
const bulk = db.collection('products').initializeUnorderedBulkOp();

bulk.insert({ sku: 'A001', name: 'Widget', qty: 100 });
bulk.insert({ sku: 'A002', name: 'Gadget', qty: 200 });
bulk.find({ sku: 'B001' }).updateOne({ $inc: { qty: -5 } });
bulk.find({ sku: 'C001', status: 'discontinued' }).deleteOne();

const result = await bulk.execute();
console.log(result);
```

## Inspecting the Result

```javascript
const result = await bulk.execute();

console.log(`Inserted:  ${result.nInserted}`);
console.log(`Updated:   ${result.nModified}`);
console.log(`Upserted:  ${result.nUpserted}`);
console.log(`Deleted:   ${result.nRemoved}`);
console.log(`Has errors: ${result.hasWriteErrors()}`);

if (result.hasWriteErrors()) {
  result.getWriteErrors().forEach(err => {
    console.error(`Error at index ${err.index}: ${err.errmsg}`);
  });
}
```

## Ordered vs. Unordered: When to Choose Unordered

| Scenario | Use Ordered | Use Unordered |
|---|---|---|
| Insert documents that may have duplicates | Yes | No |
| Operations depend on previous results | Yes | No |
| Maximize throughput, operations are independent | No | Yes |
| Partial success is acceptable | No | Yes |

## Adding upsert Operations

```javascript
const bulk = db.collection('users').initializeUnorderedBulkOp();

const users = [
  { email: 'alice@example.com', name: 'Alice' },
  { email: 'bob@example.com', name: 'Bob' }
];

users.forEach(user => {
  bulk.find({ email: user.email })
      .upsert()
      .updateOne({ $set: user });
});

await bulk.execute();
```

## Handling Errors in Unordered Mode

In unordered mode, errors do not stop execution. Always check for write errors after `execute()`:

```javascript
let result;
try {
  result = await bulk.execute();
} catch (err) {
  // A thrown error means the batch itself failed to send
  console.error('Bulk execution failed:', err.message);
  return;
}

if (result.hasWriteErrors()) {
  const errors = result.getWriteErrors();
  console.warn(`${errors.length} operations failed out of ${result.nInserted + result.nModified}`);
  errors.forEach(e => console.error(`  Index ${e.index}: ${e.errmsg}`));
}
```

## Comparing with the Modern bulkWrite API

`initializeUnorderedBulkOp()` is the legacy fluent builder API. The modern equivalent uses `bulkWrite()` with `ordered: false`:

```javascript
// Modern equivalent
await collection.bulkWrite([
  { insertOne: { document: { sku: 'A001', qty: 100 } } },
  { updateOne: { filter: { sku: 'B001' }, update: { $inc: { qty: -5 } } } }
], { ordered: false });
```

Both approaches send operations as an unordered batch. Prefer the `bulkWrite()` API for new code as it is more explicit and widely documented.

## Summary

`initializeUnorderedBulkOp()` is MongoDB's legacy fluent API for building unordered bulk write batches where operations execute independently. It maximizes write throughput by not halting on errors, making it suitable for large, independent batch jobs. Always inspect `result.getWriteErrors()` after execution to detect partial failures, and prefer the modern `bulkWrite({ ordered: false })` API for new development.
