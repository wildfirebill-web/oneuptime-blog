# How to Use the delete Command in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Delete, Write, Cleanup

Description: Learn how to delete documents in MongoDB using deleteOne, deleteMany, findOneAndDelete, and drop operations with safe patterns for production use.

---

## deleteOne

`deleteOne()` removes the first document matching the filter:

```javascript
// Delete one document by _id
const result = db.orders.deleteOne({ _id: new ObjectId("507f1f77bcf86cd799439011") });
print(`Deleted: ${result.deletedCount}`); // 0 or 1

// Delete by a unique field
db.sessions.deleteOne({ token: "abc123" });
```

## deleteMany

`deleteMany()` removes all documents matching the filter:

```javascript
// Delete all cancelled orders older than 90 days
const cutoff = new Date();
cutoff.setDate(cutoff.getDate() - 90);

const result = db.orders.deleteMany({
  status: "cancelled",
  createdAt: { $lt: cutoff }
});

print(`Deleted ${result.deletedCount} old cancelled orders`);
```

## Deleting All Documents in a Collection

```javascript
// Delete all documents but keep the collection and indexes
db.tempData.deleteMany({});

// Faster alternative - drop and recreate
db.tempData.drop();
```

For large collections, `drop()` is significantly faster than `deleteMany({})`.

## findOneAndDelete

Returns the deleted document - useful when you need the data before deletion:

```javascript
const deleted = db.jobs.findOneAndDelete(
  { status: "pending" },
  { sort: { priority: -1, createdAt: 1 } }
);

if (deleted) {
  print(`Processing job: ${deleted._id}`);
  // process the job...
}
```

This pattern is useful for queue processing.

## Soft Deletes

Many applications prefer soft deletes - marking documents as deleted rather than removing them:

```javascript
db.users.updateOne(
  { _id: "u-001" },
  {
    $set: {
      deletedAt: new Date(),
      deletedBy: "admin"
    }
  }
);

// Filter out soft-deleted documents in queries
db.users.find({ deletedAt: { $exists: false } });
```

## Deleting Inside a Transaction

```javascript
const session = db.getMongo().startSession();
session.startTransaction();

try {
  db.orders.deleteOne({ _id: "ORD-100" }, { session });
  db.inventory.updateOne(
    { productId: "P-1" },
    { $inc: { reserved: -1 } },
    { session }
  );
  session.commitTransaction();
} catch (err) {
  session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

## Write Concern for Deletes

```javascript
db.logs.deleteMany(
  { ts: { $lt: new Date("2024-01-01") } },
  { writeConcern: { w: "majority" } }
);
```

## Preventing Accidental Mass Deletes

Add a safety check before bulk deletes:

```javascript
const filter = { status: "cancelled", createdAt: { $lt: new Date("2024-01-01") } };

// Count first
const count = db.orders.countDocuments(filter);
print(`About to delete ${count} documents. Continue? (update script to proceed)`);

// Uncomment to execute:
// db.orders.deleteMany(filter);
```

## Checking Deletion Results

```javascript
const result = db.orders.deleteMany({ status: "test" });
if (result.deletedCount === 0) {
  print("No documents matched the filter");
} else {
  print(`Successfully deleted ${result.deletedCount} documents`);
}
```

## Summary

Use `deleteOne()` for targeted single-document removal and `deleteMany()` for bulk cleanup. Prefer `findOneAndDelete()` when you need the removed data (job queues, dequeue patterns). For production safety, count matching documents before bulk deletions and consider soft deletes with a `deletedAt` timestamp when auditability is required.
