# How to Create a Unique Index in MongoDB to Enforce Uniqueness

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Unique Index, Data Integrity, Database

Description: Learn how to create unique indexes in MongoDB to enforce that no two documents share the same field value, ensuring data integrity at the database level.

---

## Overview

A unique index in MongoDB enforces that the indexed field(s) contain unique values across all documents in the collection. Attempting to insert or update a document with a duplicate value on a uniquely indexed field results in an error. Unique indexes are essential for fields like email addresses, usernames, and external IDs where duplicates are not allowed.

## Creating a Unique Index

Add the `unique: true` option to `createIndex()`:

```javascript
db.users.createIndex({ email: 1 }, { unique: true })
```

Now inserting a duplicate email throws a `DuplicateKey` error:

```javascript
db.users.insertOne({ email: "alice@example.com", name: "Alice" })
// Works fine - first insert

db.users.insertOne({ email: "alice@example.com", name: "Alice2" })
// Error: E11000 duplicate key error collection: mydb.users index: email_1
```

## Unique Compound Index

Enforce uniqueness across a combination of fields:

```javascript
db.subscriptions.createIndex(
  { userId: 1, planId: 1 },
  { unique: true }
)
```

This allows the same `userId` to appear multiple times and the same `planId` to appear multiple times - but not the same `(userId, planId)` pair.

## Unique Index on Embedded Fields

```javascript
db.profiles.createIndex(
  { "contact.email": 1 },
  { unique: true }
)
```

## Handling null Values in Unique Indexes

By default, a unique index treats `null` and missing fields as a value - meaning only one document can have `null` or be missing the indexed field. To allow multiple documents without the field, use a sparse unique index or a partial unique index:

```javascript
// Sparse unique index - null/missing documents are not indexed
db.users.createIndex(
  { phone: 1 },
  { unique: true, sparse: true }
)

// Partial unique index - only index documents where phone exists
db.users.createIndex(
  { phone: 1 },
  {
    unique: true,
    partialFilterExpression: { phone: { $exists: true } }
  }
)
```

The partial index approach is more explicit and recommended over sparse for this use case.

## Creating a Unique Index on an Existing Collection

Before creating a unique index on an existing collection, check for existing duplicates - the index creation will fail if duplicates exist:

```javascript
// Find duplicate email values
db.users.aggregate([
  { $group: { _id: "$email", count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
])
```

After resolving duplicates, create the index:

```javascript
db.users.createIndex({ email: 1 }, { unique: true })
```

## Error Handling for Duplicate Key Errors

Handle duplicate key errors in application code:

```javascript
try {
  await db.collection("users").insertOne({ email: "user@example.com" })
} catch (err) {
  if (err.code === 11000) {
    console.log("Email already registered:", err.keyValue)
    // Return appropriate error response to the client
  } else {
    throw err
  }
}
```

The `err.keyValue` property contains the duplicate field and value.

## Using upsert with Unique Indexes

Use `updateOne` with `upsert: true` to safely insert-or-update:

```javascript
db.users.updateOne(
  { email: "user@example.com" },
  {
    $set: { name: "Updated Name", updatedAt: new Date() },
    $setOnInsert: { createdAt: new Date() }
  },
  { upsert: true }
)
```

This avoids duplicate key errors on concurrent inserts by acting as an atomic insert-if-not-exists.

## Unique Index vs Application-Level Validation

```text
Why prefer unique indexes over application-level uniqueness checks:
- Application checks have race conditions (two requests pass check simultaneously)
- Unique indexes provide atomic, database-level enforcement
- No extra query needed before insert - let the database enforce it
- Consistent enforcement even for direct database writes
```

## Summary

Unique indexes in MongoDB enforce data integrity by preventing duplicate values on indexed fields at the database level. They are more reliable than application-level uniqueness checks because they eliminate race conditions. Use sparse or partial unique indexes to allow multiple documents with null or missing values while still enforcing uniqueness among populated values. Always check for existing duplicates before adding a unique index to an existing collection.
