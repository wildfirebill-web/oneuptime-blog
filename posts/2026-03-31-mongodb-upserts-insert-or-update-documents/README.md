# How to Use Upserts in MongoDB to Insert or Update Documents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Upsert, updateOne, Insert, Write Operation

Description: Learn how MongoDB upserts combine insert and update into one atomic operation, eliminating race conditions in find-then-write patterns.

---

## What Is an Upsert

An upsert is a write operation that either updates an existing document if the filter matches, or inserts a new document if no match is found. MongoDB supports upserts via the `upsert: true` option on `updateOne()`, `updateMany()`, and `replaceOne()`.

Without upserts, a common pattern is:

```javascript
const existing = await collection.findOne(filter);
if (existing) {
  await collection.updateOne(filter, update);
} else {
  await collection.insertOne(newDoc);
}
```

This has a race condition between the read and write. An upsert handles both steps atomically.

## Basic Upsert with updateOne()

```javascript
await db.collection("userStats").updateOne(
  { userId: "u42" },
  {
    $set: { lastSeen: new Date() },
    $inc: { loginCount: 1 }
  },
  { upsert: true }
)
```

If a document with `userId: "u42"` exists, `lastSeen` and `loginCount` are updated. If not, a new document is created with `userId: "u42"`, `lastSeen`, and `loginCount: 1`.

## Checking the Upsert Result

```javascript
const result = await db.collection("settings").updateOne(
  { key: "maxConnections" },
  { $setOnInsert: { key: "maxConnections", value: 100, createdAt: new Date() } },
  { upsert: true }
);

if (result.upsertedCount > 0) {
  console.log("New document created:", result.upsertedId);
} else {
  console.log("Existing document matched:", result.matchedCount);
}
```

## Using $setOnInsert

`$setOnInsert` only applies fields when the operation inserts (not updates). This is useful for setting default values or `createdAt` timestamps only on new documents:

```javascript
await db.collection("profiles").updateOne(
  { userId: "u99" },
  {
    $set: { lastUpdated: new Date() },
    $setOnInsert: { createdAt: new Date(), tier: "free" }
  },
  { upsert: true }
)
```

If the document already exists, only `lastUpdated` changes. If it is new, `createdAt` and `tier` are also set.

## Upsert with replaceOne()

```javascript
await db.collection("cache").replaceOne(
  { key: "homepage_data" },
  { key: "homepage_data", data: fetchedData, cachedAt: new Date() },
  { upsert: true }
)
```

If the cache entry exists it is fully replaced; otherwise a new one is inserted.

## Upsert Pitfall: Filter Fields in New Documents

When MongoDB creates a new document via upsert, it includes the filter fields in the inserted document. Make sure your filter fields are correct - they become part of the new record:

```javascript
// New doc will include: { _id: ..., userId: "u42", lastSeen: ..., loginCount: 1 }
await collection.updateOne(
  { userId: "u42" },
  { $set: { lastSeen: new Date() }, $inc: { loginCount: 1 } },
  { upsert: true }
)
```

## Common Mistakes

- Not using `$setOnInsert` when you need `createdAt` only on new documents.
- Using upsert with `updateMany()` - if no match is found, only one document is inserted.
- Forgetting that filter fields are embedded in the new document.

## Summary

MongoDB upserts atomically handle the insert-or-update pattern using `{ upsert: true }` on update operations. Use `$setOnInsert` to apply fields only when a new document is created. Check `upsertedCount` and `upsertedId` in the result to distinguish inserts from updates.
