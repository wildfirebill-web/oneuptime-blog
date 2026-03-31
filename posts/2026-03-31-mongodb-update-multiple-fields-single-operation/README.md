# How to Update Multiple Fields in a Single Operation in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Operator, Atomic, Performance

Description: Learn how to update multiple fields in a single MongoDB operation using $set, combining multiple update operators, and why single operations are preferred.

---

Updating multiple fields in one database call is more efficient than issuing separate update commands. MongoDB's update operators support combining multiple field operations in a single `updateOne` or `updateMany` call.

## Updating Multiple Fields with $set

Pass multiple field-value pairs to `$set` to update them all atomically:

```javascript
db.users.updateOne(
  { _id: userId },
  {
    $set: {
      firstName: "Jane",
      lastName: "Doe",
      email: "jane.doe@example.com",
      updatedAt: new Date()
    }
  }
)
```

All specified fields are updated in a single atomic write.

## Combining Multiple Update Operators

You can mix different operators in one update:

```javascript
db.products.updateOne(
  { _id: productId },
  {
    $set: {
      name: "Widget Pro",
      category: "electronics"
    },
    $inc: {
      viewCount: 1,
      stockLevel: -1
    },
    $unset: {
      legacyField: ""
    },
    $currentDate: {
      lastModified: true
    }
  }
)
```

This single operation:
- Sets `name` and `category`
- Increments `viewCount` by 1 and decrements `stockLevel` by 1
- Removes `legacyField`
- Sets `lastModified` to the current date

## Why Single Operations Are Better

Using one operation instead of multiple is preferable for several reasons:

1. Atomicity: all changes happen together - no partial states visible to other reads
2. Performance: one network round-trip instead of N
3. Reduced locking time on the document

```javascript
// LESS IDEAL - three separate round-trips
db.users.updateOne({ _id: id }, { $set: { name: "Jane" } });
db.users.updateOne({ _id: id }, { $set: { email: "jane@example.com" } });
db.users.updateOne({ _id: id }, { $currentDate: { updatedAt: true } });

// BETTER - single atomic operation
db.users.updateOne(
  { _id: id },
  {
    $set: { name: "Jane", email: "jane@example.com" },
    $currentDate: { updatedAt: true }
  }
)
```

## Using $setOnInsert for Upserts

When doing an upsert, use `$setOnInsert` for fields that should only be set when inserting:

```javascript
db.users.updateOne(
  { email: "jane@example.com" },
  {
    $set: { name: "Jane Doe", lastLogin: new Date() },
    $setOnInsert: {
      createdAt: new Date(),
      role: "user",
      isActive: true
    }
  },
  { upsert: true }
)
```

`$set` runs on both insert and update; `$setOnInsert` only runs on insert.

## Bulk Updates Across Many Documents

Update multiple fields on all matching documents:

```javascript
db.orders.updateMany(
  { status: "pending", createdAt: { $lt: new Date("2024-01-01") } },
  {
    $set: {
      status: "expired",
      expiredAt: new Date()
    },
    $inc: { processingAttempts: 1 }
  }
)
```

## Conditional Updates with $set and Aggregation Pipeline

For update logic based on current field values, use an aggregation pipeline update (MongoDB 4.2+):

```javascript
db.products.updateOne(
  { _id: productId },
  [
    {
      $set: {
        salePrice: { $multiply: ["$price", 0.9] },
        onSale: true,
        updatedAt: "$$NOW"
      }
    }
  ]
)
```

## Return the Updated Document

Use `findOneAndUpdate` to return the document after the update:

```javascript
const updated = await db.collection("users").findOneAndUpdate(
  { _id: new ObjectId(userId) },
  {
    $set: { name: "Jane", email: "jane@example.com" },
    $currentDate: { updatedAt: true }
  },
  { returnDocument: "after" }
);
```

## Summary

Use a single `updateOne` or `updateMany` call with multiple fields in one `$set` object to update several fields atomically and efficiently. Combine `$set`, `$inc`, `$unset`, and other operators in the same update document. For upsert scenarios, use `$setOnInsert` for fields that should only populate on document creation. Single-operation updates reduce network round-trips and ensure atomicity.
