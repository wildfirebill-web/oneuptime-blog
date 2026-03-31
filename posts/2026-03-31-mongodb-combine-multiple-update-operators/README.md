# How to Combine Multiple Update Operators in a Single MongoDB Update

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Operator, Atomic, Write

Description: Learn how to combine $set, $inc, $push, $pull, $unset and other operators in one MongoDB update call for efficient, atomic document modifications.

---

## Overview

MongoDB update documents support multiple operators simultaneously. Combining operators in a single update reduces round-trips to the database and ensures all changes are applied atomically. Any number of different operator types can be combined in one update call.

## Operators You Can Combine

| Operator    | Purpose                                      |
|-------------|----------------------------------------------|
| `$set`      | Set field values                             |
| `$unset`    | Remove fields                                |
| `$inc`      | Increment/decrement numeric fields           |
| `$push`     | Append to arrays                             |
| `$pull`     | Remove matching elements from arrays         |
| `$addToSet` | Append to array only if value is not present |
| `$rename`   | Rename a field                               |
| `$mul`      | Multiply a numeric field                     |

## Basic Multi-Operator Example

```javascript
// Update an order: set status, increment version, push history entry
db.orders.updateOne(
  { _id: orderId },
  {
    $set: {
      status: 'shipped',
      shippedAt: new Date(),
    },
    $inc: {
      version: 1,
      'metrics.shipCount': 1,
    },
    $push: {
      history: { event: 'shipped', at: new Date() },
    },
  }
);
```

All three operators execute together in a single write - there is no risk of a partial update where status is updated but history is not.

## Combining $set and $unset

A common schema migration pattern: set new fields and remove deprecated ones in the same update:

```javascript
db.users.updateMany(
  { legacyField: { $exists: true } },
  {
    $set:   { newField: 'default' },
    $unset: { legacyField: '' },
  }
);
```

## Combining $inc with $push and $pull

```javascript
// When a user completes a course: increment completed count,
// push to completedCourses, and pull from inProgressCourses
db.users.updateOne(
  { _id: userId, 'inProgressCourses.courseId': courseId },
  {
    $inc: { completedCount: 1 },
    $push: {
      completedCourses: {
        courseId,
        completedAt: new Date(),
        score,
      },
    },
    $pull: {
      inProgressCourses: { courseId },
    },
  }
);
```

## Combining $set, $addToSet, and $inc for Tag Systems

```javascript
// Add a tag to an article and increment the tag's view count
db.articles.updateOne(
  { _id: articleId },
  {
    $addToSet: { tags: 'mongodb' },
    $inc:      { viewCount: 1 },
    $set:      { updatedAt: new Date() },
  }
);
```

## Restrictions on Combining Operators

You cannot update the same field path with two different operators in a single update:

```javascript
// This will FAIL - cannot $set and $inc the same field
db.items.updateOne(
  { _id: id },
  {
    $set: { count: 5 },
    $inc: { count: 1 }, // Error: conflicting paths
  }
);
```

Likewise, you cannot use `$push` and `$pull` on the same array in the same update - use two sequential updates or a pipeline update for that.

## Using returnDocument to Verify Results

```javascript
const result = await db.collection('orders').findOneAndUpdate(
  { _id: orderId },
  {
    $set: { status: 'completed', completedAt: new Date() },
    $inc: { 'stats.completedOrders': 1 },
    $push: { auditLog: { action: 'completed', at: new Date() } },
  },
  { returnDocument: 'after' }
);

console.log('Updated order:', result);
```

## Summary

Combining multiple update operators in a single MongoDB update is both more efficient and safer than multiple sequential updates. The changes are applied atomically, eliminating the risk of partial updates. The key constraint is that no two operators can target the same field path. Use multi-operator updates for status transitions, schema migrations, and any write that involves multiple field types changing together.
