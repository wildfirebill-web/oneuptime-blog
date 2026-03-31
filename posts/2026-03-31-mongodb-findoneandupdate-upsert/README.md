# How to Use findOneAndUpdate with Upsert in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, findOneAndUpdate, Upsert, Update, Insert

Description: Learn how to use findOneAndUpdate with upsert:true to atomically insert or update documents in MongoDB, avoiding duplicate key races.

---

## Overview

The `upsert` option in `findOneAndUpdate` instructs MongoDB to insert a new document if no document matches the filter. This is an atomic operation - it eliminates the race condition that exists when doing a separate find-then-insert pattern.

## Basic Upsert Syntax

```javascript
const result = await db.collection('users').findOneAndUpdate(
  { email: 'alice@example.com' },       // filter
  { $set: { lastSeen: new Date() } },   // update
  { upsert: true, returnDocument: 'after' }
);
```

If `alice@example.com` exists, her `lastSeen` is updated. If she does not exist, MongoDB inserts a new document with `email: 'alice@example.com'` and `lastSeen: new Date()`.

## Using $setOnInsert for Insert-Only Fields

`$setOnInsert` applies fields only when the document is being inserted, not when it is being updated:

```javascript
const result = await db.collection('users').findOneAndUpdate(
  { email: 'bob@example.com' },
  {
    $set: { lastSeen: new Date() },
    $setOnInsert: {
      createdAt: new Date(),
      role: 'member',
      emailVerified: false,
    },
  },
  { upsert: true, returnDocument: 'after' }
);
```

`createdAt`, `role`, and `emailVerified` are only written when a new document is created. Existing documents are not modified on those fields.

## Detecting Insert vs Update

The returned result includes `lastErrorObject` which tells you whether an insert or update occurred:

```javascript
// Using the raw driver (not Mongoose)
const result = await db.collection('users').findOneAndUpdate(
  { email: 'carol@example.com' },
  { $set: { lastSeen: new Date() }, $setOnInsert: { createdAt: new Date() } },
  { upsert: true, returnDocument: 'after', includeResultMetadata: true }
);

if (result.lastErrorObject.upserted) {
  console.log('New user created:', result.lastErrorObject.upserted);
} else {
  console.log('Existing user updated');
}
```

## Upsert with Compound Filters

Compound filters are commonly used to scope upserts to unique business keys:

```javascript
// Upsert a daily analytics record
await db.collection('analytics').findOneAndUpdate(
  {
    date: new Date().toISOString().split('T')[0],
    page: '/home',
  },
  {
    $inc: { views: 1 },
    $setOnInsert: {
      date: new Date().toISOString().split('T')[0],
      page: '/home',
      firstView: new Date(),
    },
  },
  { upsert: true }
);
```

## Creating a Unique Index to Prevent Races

Even with upsert, you should create a unique index on the filter key to prevent duplicate documents under extremely high concurrency:

```javascript
db.users.createIndex({ email: 1 }, { unique: true });
```

If two concurrent upserts race on the same email, the unique index ensures only one succeeds. The other will get a `DuplicateKeyError` (error code 11000), which you should handle:

```javascript
try {
  await db.collection('users').findOneAndUpdate(
    { email },
    { $setOnInsert: { email, createdAt: new Date() } },
    { upsert: true, returnDocument: 'after' }
  );
} catch (err) {
  if (err.code === 11000) {
    // Another request created this user first - just fetch them
    return db.collection('users').findOne({ email });
  }
  throw err;
}
```

## Mongoose findOneAndUpdate with Upsert

```javascript
const user = await User.findOneAndUpdate(
  { email },
  { $set: { lastSeen: new Date() }, $setOnInsert: { role: 'member' } },
  { upsert: true, new: true, setDefaultsOnInsert: true }
);
```

The `setDefaultsOnInsert: true` option applies Mongoose schema defaults when creating a new document.

## Summary

`findOneAndUpdate` with `upsert: true` provides a race-condition-free way to insert or update documents in a single atomic operation. Use `$setOnInsert` for fields that should only be written on creation. Protect upsert operations with a unique index on the filter key to handle extreme concurrency, and catch `DuplicateKeyError` gracefully as a fallback.
