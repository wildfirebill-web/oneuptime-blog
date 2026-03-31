# How to Use $setOnInsert to Set Default Values During Upserts in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Database, Update

Description: Learn how MongoDB's $setOnInsert operator populates fields only when an upsert creates a new document, making it ideal for setting immutable defaults.

---

When running an upsert in MongoDB, you sometimes want to set certain fields only when a new document is created - not when an existing document is updated. This is exactly what `$setOnInsert` is for. Fields specified inside `$setOnInsert` are written only on insert; if the document already exists and is merely updated, those fields are ignored.

## Syntax

```javascript
db.collection.updateOne(
  { filter },
  {
    $set: { ... },
    $setOnInsert: { fieldName: value }
  },
  { upsert: true }
)
```

`$setOnInsert` has no effect unless `upsert: true` is specified and the filter matches no existing document.

## Basic Example - Setting a Created Timestamp

```javascript
db.users.updateOne(
  { email: "alice@example.com" },
  {
    $set: { lastLogin: new Date() },
    $setOnInsert: { createdAt: new Date(), role: "viewer" }
  },
  { upsert: true }
)
```

- **New document**: `createdAt` and `role` are set alongside `lastLogin`.
- **Existing document**: only `lastLogin` is updated; `createdAt` and `role` are untouched.

## Preventing Overwrites of Immutable Fields

Consider a counter document that must keep its original `createdBy` field:

```javascript
db.counters.updateOne(
  { name: "pageViews" },
  {
    $inc: { count: 1 },
    $setOnInsert: { createdBy: "system", initialValue: 0 }
  },
  { upsert: true }
)
```

Every increment call is safe - `createdBy` and `initialValue` only appear when the counter is first created.

## Verifying Insert vs. Update Behavior

The result object from the driver tells you whether an insert or update occurred:

```javascript
const result = await db.collection("products").updateOne(
  { sku: "ABC-123" },
  {
    $set: { price: 29.99 },
    $setOnInsert: { createdAt: new Date(), inventory: 100 }
  },
  { upsert: true }
)

console.log("Matched:", result.matchedCount)
console.log("Modified:", result.modifiedCount)
console.log("UpsertedId:", result.upsertedId)
// UpsertedId is non-null only on insert
```

## Combining with $currentDate

You can pair `$setOnInsert` with `$currentDate` for server-side timestamps:

```javascript
db.profiles.updateOne(
  { userId: "u_789" },
  {
    $set: { displayName: "Bob" },
    $setOnInsert: { tier: "free" },
    $currentDate: { updatedAt: true }
  },
  { upsert: true }
)
```

On insert, all three operators fire. On update, only `$set` and `$currentDate` apply.

## Common Mistakes

**Forgetting upsert: true** - `$setOnInsert` silently does nothing without the upsert flag:

```javascript
// This update runs fine but $setOnInsert has no effect
db.items.updateOne(
  { _id: 1 },
  { $setOnInsert: { createdAt: new Date() } }
  // missing { upsert: true }
)
```

**Using $setOnInsert for mutable fields** - Fields that should change on every update belong in `$set`, not `$setOnInsert`.

## Summary

`$setOnInsert` populates fields exclusively when an upsert creates a new document, leaving existing documents unchanged. It is ideal for recording creation timestamps, assigning default roles, and protecting immutable fields from accidental overwrites during routine updates. Always pair it with `upsert: true` or the operator has no effect.
