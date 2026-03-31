# How to Delete a Single Document in MongoDB with deleteOne()

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, deleteOne, Delete, Write Operation, Document

Description: Learn how to safely delete a single document in MongoDB using deleteOne(), with filter strategies and result verification.

---

## What Is deleteOne()

`deleteOne()` removes the first document that matches a given filter from a collection. If the filter matches multiple documents, only the first one (by natural order or index order) is deleted. No other documents are affected.

Signature:

```javascript
db.collection.deleteOne(filter, options)
```

## Basic Example

```javascript
db.users.deleteOne({ email: "alice@example.com" })
```

This deletes the first user document where `email` equals `"alice@example.com"`.

## Deleting by _id (Safest Approach)

The safest way to delete a specific document is to filter by its unique `_id`:

```javascript
const { ObjectId } = require("mongodb");

await db.collection("orders").deleteOne({
  _id: new ObjectId("64a1b2c3d4e5f6789abcdef0")
});
```

Filtering by `_id` guarantees exactly one document is targeted.

## Checking the Result

`deleteOne()` returns a result object with a `deletedCount` property:

```javascript
const result = await db.collection("sessions").deleteOne({
  token: "abc123xyz"
});

if (result.deletedCount === 1) {
  console.log("Session deleted successfully");
} else {
  console.log("Session not found");
}
```

`deletedCount` is `1` if a document was deleted, `0` if none matched the filter.

## Using deleteOne() Inside a Transaction

For operations that must be atomic across multiple steps:

```javascript
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    await db.collection("carts").deleteOne({ userId: "u42" }, { session });
    await db.collection("reservations").deleteMany({ userId: "u42" }, { session });
  });
} finally {
  await session.endSession();
}
```

## deleteOne() vs findOneAndDelete()

| Method | Returns |
|--------|---------|
| `deleteOne()` | Result object with `deletedCount` |
| `findOneAndDelete()` | The deleted document itself |

Use `findOneAndDelete()` when you need the document's data after deletion (e.g., to log it or return it to the caller).

## Common Mistakes

- Not filtering by a unique field when you intend to delete a specific record.
- Passing an empty filter `{}` - this is valid but deletes one arbitrary document.
- Not checking `deletedCount` to confirm the document existed.

## Summary

`deleteOne()` removes the first document matching the filter and returns a `deletedCount` indicating success. For safety, filter by `_id` or another unique field. Check the result to confirm deletion, and use `findOneAndDelete()` when you also need the document data returned.
