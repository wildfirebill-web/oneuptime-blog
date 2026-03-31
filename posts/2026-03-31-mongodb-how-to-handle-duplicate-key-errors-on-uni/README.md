# How to Handle Duplicate Key Errors on Unique Indexes in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Unique Index, Error Handling, Database

Description: Learn how to detect, handle, and prevent duplicate key errors (E11000) on unique indexes in MongoDB, with strategies for both write operations and data migration.

---

## Overview

A duplicate key error (error code `11000`) in MongoDB occurs when an insert or update operation violates a unique index constraint. Understanding how to handle these errors gracefully in application code, how to prevent them with the right write strategies, and how to resolve existing duplicates before creating a unique index is essential for building reliable MongoDB applications.

## Understanding the E11000 Error

When a unique constraint is violated, MongoDB throws an error with code `11000`:

```text
E11000 duplicate key error collection: mydb.users index: email_1
dup key: { email: "alice@example.com" }
```

The error includes:
- The collection name
- The index that was violated
- The duplicate key value

## Handling E11000 in Application Code

### Node.js (with MongoDB Driver)

```javascript
async function createUser(userData) {
  try {
    const result = await db.collection("users").insertOne(userData)
    return { success: true, id: result.insertedId }
  } catch (err) {
    if (err.code === 11000) {
      const field = Object.keys(err.keyPattern)[0]
      return {
        success: false,
        error: `${field} already exists`,
        field: field
      }
    }
    throw err
  }
}
```

### Node.js (with Mongoose)

```javascript
async function saveUser(userData) {
  try {
    const user = new User(userData)
    await user.save()
    return user
  } catch (err) {
    if (err.name === "MongoServerError" && err.code === 11000) {
      const field = Object.keys(err.keyValue)[0]
      throw new Error(`A user with this ${field} already exists`)
    }
    throw err
  }
}
```

## Using upsert to Avoid Duplicate Key Errors

Use `updateOne` with `upsert: true` for idempotent insert-or-update:

```javascript
const result = await db.collection("users").updateOne(
  { email: "alice@example.com" },
  {
    $set: { name: "Alice", updatedAt: new Date() },
    $setOnInsert: { createdAt: new Date(), role: "user" }
  },
  { upsert: true }
)

// result.upsertedId is set if a new document was inserted
// result.matchedCount > 0 if an existing document was updated
```

`$setOnInsert` only applies when a new document is created, not on updates.

## Bulk Operations and Handling Partial Failures

When inserting multiple documents, use `ordered: false` to continue after duplicate errors:

```javascript
const docs = [
  { email: "a@example.com", name: "Alice" },
  { email: "b@example.com", name: "Bob" },
  { email: "a@example.com", name: "Alice2" }  // duplicate
]

try {
  const result = await db.collection("users").insertMany(docs, { ordered: false })
  console.log(`Inserted ${result.insertedCount} documents`)
} catch (err) {
  if (err.code === 11000 || err.writeErrors) {
    const inserted = err.result?.nInserted || 0
    const errors = err.writeErrors || []
    console.log(`Inserted ${inserted}, ${errors.length} duplicates skipped`)
  }
}
```

With `ordered: false`, non-duplicate documents are inserted even if others fail.

## Finding and Resolving Existing Duplicates

Before adding a unique index to an existing collection, identify and remove duplicates:

```javascript
// Step 1: Find duplicates
db.users.aggregate([
  { $group: { _id: "$email", count: { $sum: 1 }, ids: { $push: "$_id" } } },
  { $match: { count: { $gt: 1 } } }
])
```

```javascript
// Step 2: Keep one document per duplicate group, delete the rest
db.users.aggregate([
  { $group: { _id: "$email", ids: { $push: "$_id" }, count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
]).forEach(group => {
  const idsToDelete = group.ids.slice(1)  // Keep the first, delete the rest
  db.users.deleteMany({ _id: { $in: idsToDelete } })
})
```

## Preventing Duplicates with findOneAndUpdate

Use `findOneAndUpdate` with `upsert` for atomic check-and-insert:

```javascript
const result = await db.collection("users").findOneAndUpdate(
  { email: "alice@example.com" },
  { $setOnInsert: { email: "alice@example.com", name: "Alice", createdAt: new Date() } },
  { upsert: true, returnDocument: "after" }
)
```

## Extracting Field Info from the Error

Parse the duplicate field and value from the error for better error messages:

```javascript
function parseDuplicateKeyError(err) {
  if (err.code !== 11000) return null
  return {
    field: Object.keys(err.keyPattern)[0],
    value: Object.values(err.keyValue)[0],
    index: err.keyPattern
  }
}

// Usage
try {
  await collection.insertOne(doc)
} catch (err) {
  const dupInfo = parseDuplicateKeyError(err)
  if (dupInfo) {
    return res.status(409).json({
      error: "Conflict",
      message: `${dupInfo.field} '${dupInfo.value}' already exists`
    })
  }
  throw err
}
```

## Summary

Duplicate key errors on unique indexes are a normal part of working with MongoDB data integrity constraints. Handle them by catching error code `11000` in application code, using `upsert` for idempotent write operations, and setting `ordered: false` for bulk inserts to maximize throughput while reporting failures. Before creating a unique index on existing data, always audit for and resolve existing duplicates. Proper error handling makes unique index violations informative user-facing messages rather than unexpected crashes.
