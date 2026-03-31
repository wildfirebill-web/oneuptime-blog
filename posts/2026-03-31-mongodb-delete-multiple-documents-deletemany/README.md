# How to Delete Multiple Documents in MongoDB with deleteMany()

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, deleteMany, Delete, Bulk Delete, Write Operation

Description: Learn how to use deleteMany() to remove all documents matching a filter in MongoDB, including safety tips and result verification.

---

## What Is deleteMany()

`deleteMany()` removes every document in a collection that matches the specified filter. It is the bulk counterpart to `deleteOne()` and is commonly used for cleanup tasks, expiring records, or removing groups of related data.

Signature:

```javascript
db.collection.deleteMany(filter, options)
```

## Basic Example

Delete all expired sessions:

```javascript
db.sessions.deleteMany({ expiresAt: { $lt: new Date() } })
```

Every session document where `expiresAt` is in the past is removed.

## Deleting All Documents in a Collection

Passing an empty filter deletes every document but keeps the collection and its indexes:

```javascript
await db.collection("tempLogs").deleteMany({})
```

This is different from `drop()`, which removes the collection and its indexes entirely.

## Checking the Result

```javascript
const result = await db.collection("notifications").deleteMany({
  read: true,
  createdAt: { $lt: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) }
});

console.log(`Deleted ${result.deletedCount} notifications`);
```

## Combining Multiple Conditions

Use query operators to build precise filters:

```javascript
db.users.deleteMany({
  $and: [
    { status: "inactive" },
    { lastLogin: { $lt: new Date("2023-01-01") } },
    { role: { $ne: "admin" } }
  ]
})
```

## Batching Large Deletes

Deleting millions of documents in one call can impact performance. Batch the work:

```javascript
async function deleteBatch(collection, filter, batchSize = 1000) {
  let totalDeleted = 0;
  let deleted;
  do {
    const ids = await collection
      .find(filter, { projection: { _id: 1 } })
      .limit(batchSize)
      .toArray();
    if (ids.length === 0) break;
    const result = await collection.deleteMany({
      _id: { $in: ids.map(d => d._id) }
    });
    deleted = result.deletedCount;
    totalDeleted += deleted;
  } while (deleted === batchSize);
  return totalDeleted;
}
```

## deleteMany() vs drop()

| Action | Keeps collection | Keeps indexes | Speed (large collections) |
|--------|-----------------|---------------|--------------------------|
| `deleteMany({})` | Yes | Yes | Slow |
| `collection.drop()` | No | No | Fast |

If you need to remove all data and recreate the collection, `drop()` followed by index creation is much faster.

## Common Mistakes

- Using an empty filter `{}` accidentally - always double-check your filter before running.
- Not indexing the filter field, causing a full collection scan on large collections.
- Skipping result verification when the number of deleted documents is important.

## Summary

`deleteMany()` is the right tool for removing groups of MongoDB documents that match a condition. Always use a specific filter rather than `{}` unless you intend to clear the entire collection, and check `deletedCount` in the result. For large deletions, batch the operation to reduce lock contention and performance impact.
