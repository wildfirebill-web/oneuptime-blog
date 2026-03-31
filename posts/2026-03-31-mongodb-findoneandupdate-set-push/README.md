# How to Use findOneAndUpdate with $set and $push Together in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, findOneAndUpdate, Update, Array, Operator

Description: Learn how to combine $set and $push in a single findOneAndUpdate call to update fields and append to arrays atomically in MongoDB.

---

## Overview

`findOneAndUpdate` is MongoDB's atomic find-and-modify operation. It finds a document matching a filter, applies an update, and returns either the original or updated document. Combining `$set` and `$push` in a single update lets you modify scalar fields and append to arrays in one round-trip.

## Basic Syntax

```javascript
db.collection.findOneAndUpdate(
  filter,
  update,
  options
)
```

## Combining $set and $push

The update document can contain multiple operators. `$set` updates specific fields while `$push` appends elements to an array:

```javascript
// Update order status and append a status history entry
const result = await db.collection('orders').findOneAndUpdate(
  { _id: orderId, status: 'pending' },
  {
    $set: {
      status: 'processing',
      updatedAt: new Date(),
      assignedTo: 'warehouse-team',
    },
    $push: {
      history: {
        status: 'processing',
        changedAt: new Date(),
        changedBy: 'system',
      },
    },
  },
  { returnDocument: 'after' }
);
```

The `returnDocument: 'after'` option returns the document as it looks after the update is applied.

## Real-World Example: User Activity Tracking

```javascript
const mongoose = require('mongoose');

async function recordUserLogin(userId, ipAddress) {
  return User.findOneAndUpdate(
    { _id: userId, isActive: true },
    {
      $set: {
        lastLoginAt: new Date(),
        lastLoginIp: ipAddress,
      },
      $push: {
        loginHistory: {
          $each: [{ ip: ipAddress, at: new Date() }],
          $slice: -100, // Keep only the last 100 entries
        },
      },
    },
    { new: true, select: '-password' }
  );
}
```

The `$each` and `$slice` modifiers inside `$push` let you append multiple items while limiting the array length.

## Updating Nested Fields with $set

```javascript
// Update a nested field and push to an array
await db.collection('products').findOneAndUpdate(
  { sku: 'WIDGET-42' },
  {
    $set: {
      'inventory.quantity': 150,
      'inventory.lastRestocked': new Date(),
    },
    $push: {
      'inventory.restockLog': {
        quantity: 150,
        date: new Date(),
        supplier: 'ACME Corp',
      },
    },
  },
  { returnDocument: 'after', upsert: false }
);
```

## Using $addToSet Instead of $push for Uniqueness

If you want to append only when the value does not already exist, use `$addToSet` in place of `$push`:

```javascript
await db.collection('users').findOneAndUpdate(
  { _id: userId },
  {
    $set: { updatedAt: new Date() },
    $addToSet: { roles: 'editor' }, // Only adds 'editor' if not already in the array
  },
  { returnDocument: 'after' }
);
```

## Handling the Case Where No Document Matches

```javascript
const result = await db.collection('orders').findOneAndUpdate(
  { _id: orderId },
  { $set: { status: 'shipped' }, $push: { history: { status: 'shipped', at: new Date() } } },
  { returnDocument: 'after' }
);

if (!result) {
  console.log('Order not found or already in a terminal state');
}
```

`findOneAndUpdate` returns `null` when no document matches the filter (and `upsert` is not set).

## Summary

Combining `$set` and `$push` in a single `findOneAndUpdate` call allows you to atomically update document fields and append to arrays without multiple round-trips. Use `$each` with `$slice` to cap array growth, `$addToSet` when uniqueness matters, and `returnDocument: 'after'` to get the updated state back in the same operation.
