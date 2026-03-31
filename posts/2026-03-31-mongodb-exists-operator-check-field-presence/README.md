# How to Use $exists to Check for Field Presence in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $exists, Field Presence, Filter

Description: Learn how to use MongoDB's $exists operator to find documents that have or are missing a specific field, including null value behavior.

---

## What Is $exists

The `$exists` operator matches documents based on whether a specified field is present in the document. Use it to find documents with optional fields, detect schema inconsistencies, or clean up data migrations.

Syntax:

```javascript
{ field: { $exists: true } }   // field must be present
{ field: { $exists: false } }  // field must be absent
```

## Finding Documents Where a Field Exists

```javascript
// Users who have a phoneNumber field (even if null)
db.users.find({ phoneNumber: { $exists: true } })
```

This matches every document that has a `phoneNumber` key, regardless of its value (including `null`).

## Finding Documents Where a Field Is Missing

```javascript
// Users who do NOT have a phoneNumber field
db.users.find({ phoneNumber: { $exists: false } })
```

This matches documents where the `phoneNumber` key is completely absent from the document.

## $exists: true vs Querying for null

There is an important difference:

```javascript
// Matches documents where phoneNumber is null
db.users.find({ phoneNumber: null })

// Matches documents where phoneNumber is null OR the field is missing
db.users.find({ phoneNumber: null })  // Both cases match!

// Only matches documents where field is present (even if null)
db.users.find({ phoneNumber: { $exists: true } })

// Only matches documents with the field set to null explicitly
db.users.find({ phoneNumber: null, phoneNumber: { $exists: true } })
```

To find documents where a field exists and is null:

```javascript
db.users.find({
  phoneNumber: { $exists: true, $eq: null }
})
```

## Combining $exists with Other Operators

Check field presence before applying other conditions to avoid null comparisons:

```javascript
db.orders.find({
  discount: { $exists: true, $gt: 0 }
})
```

This returns orders that have a `discount` field with a positive value, skipping documents without the field.

## Schema Migration Use Case

Find documents missing a field that was added in a new schema version:

```javascript
const outdatedDocs = await db.collection("users").find({
  preferences: { $exists: false }
}).toArray();

// Backfill default values
await db.collection("users").updateMany(
  { preferences: { $exists: false } },
  { $set: { preferences: { theme: "light", notifications: true } } }
);
```

## $exists with Nested Fields

Use dot notation to check for nested field existence:

```javascript
// Find orders that have address.city set
db.orders.find({ "address.city": { $exists: true } })
```

## Index Behavior

`$exists: true` can use a sparse index, which only indexes documents that have the field:

```javascript
await db.collection("users").createIndex(
  { phoneNumber: 1 },
  { sparse: true }  // Only indexes documents where phoneNumber exists
);

// This query can use the sparse index
db.users.find({ phoneNumber: { $exists: true } })
```

## Common Mistakes

- Assuming `{ field: null }` only matches explicit null values - it also matches missing fields.
- Using `$exists: false` on indexed fields - sparse indexes do not help here.
- Forgetting that `$exists: true` matches both null and non-null values.

## Summary

`$exists` is the reliable way to check whether a field is present in a MongoDB document. Use `$exists: true` to find documents with the field (including null values) and `$exists: false` to find documents missing the field entirely. Combine it with `$eq: null` to distinguish between null values and absent fields, and use sparse indexes to optimize `$exists: true` queries.
