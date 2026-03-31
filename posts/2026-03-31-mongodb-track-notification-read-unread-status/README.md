# How to Track Notification Read/Unread Status in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Notification, Read, Unread, Counter

Description: Implement efficient read/unread tracking for notifications in MongoDB using atomic counters, partial indexes, and bulk mark-as-read operations.

---

## The Challenge

Tracking which notifications a user has read sounds simple but has two performance traps: counting unread notifications with `countDocuments` on every page load (slow at scale), and bulk-updating thousands of notifications to "read" inside a transaction (lock contention).

## Schema

```javascript
// Notification document
{
  _id: ObjectId("..."),
  userId: "user_abc123",
  message: "You have a new follower",
  type: "social",
  isRead: false,
  readAt: null,
  createdAt: new Date()
}

// Per-user state document (separate collection)
{
  userId: "user_abc123",
  unreadCount: 5,
  lastReadAt: null,
  updatedAt: new Date()
}
```

## Indexes for Read/Unread Queries

```javascript
// Efficient query for unread notifications
db.notifications.createIndex({ userId: 1, isRead: 1, createdAt: -1 });

// Partial index - only indexes unread documents (saves space and write overhead)
db.notifications.createIndex(
  { userId: 1, createdAt: -1 },
  { partialFilterExpression: { isRead: false } }
);
```

## Incrementing the Unread Counter

When creating a new notification, increment the counter atomically:

```javascript
async function createNotification(userId, message, type) {
  const session = client.startSession();
  let notifId;
  await session.withTransaction(async () => {
    const result = await db.collection("notifications").insertOne(
      { userId, message, type, isRead: false, readAt: null, createdAt: new Date() },
      { session }
    );
    notifId = result.insertedId;

    await db.collection("userNotificationState").updateOne(
      { userId },
      { $inc: { unreadCount: 1 }, $set: { updatedAt: new Date() } },
      { upsert: true, session }
    );
  });
  await session.endSession();
  return notifId;
}
```

## Mark a Single Notification as Read

```javascript
async function markOneRead(notificationId, userId) {
  const result = await db.collection("notifications").findOneAndUpdate(
    { _id: notificationId, userId, isRead: false },
    { $set: { isRead: true, readAt: new Date() } },
    { returnDocument: "after" }
  );

  if (result) {
    // Only decrement if the doc was actually updated (was unread before)
    await db.collection("userNotificationState").updateOne(
      { userId, unreadCount: { $gt: 0 } },
      { $inc: { unreadCount: -1 } }
    );
  }
}
```

## Mark All as Read (Bulk)

For bulk operations, avoid transactions by using a "last read timestamp" approach:

```javascript
async function markAllRead(userId) {
  const now = new Date();

  // Update the state document with the timestamp
  await db.collection("userNotificationState").updateOne(
    { userId },
    { $set: { unreadCount: 0, lastReadAt: now } },
    { upsert: true }
  );

  // Lazily mark notifications as read in the background
  await db.collection("notifications").updateMany(
    { userId, isRead: false, createdAt: { $lte: now } },
    { $set: { isRead: true, readAt: now } }
  );
}
```

The state document update is instant (O(1)). The background `updateMany` can run asynchronously.

## Get Unread Count Without Counting

```javascript
// O(1) lookup using the counter
async function getUnreadCount(userId) {
  const state = await db
    .collection("userNotificationState")
    .findOne({ userId }, { projection: { unreadCount: 1 } });
  return state?.unreadCount ?? 0;
}
```

## Periodic Counter Reconciliation

Counters can drift due to bugs or crashes. Reconcile periodically:

```javascript
async function reconcileUnreadCount(userId) {
  const actualCount = await db
    .collection("notifications")
    .countDocuments({ userId, isRead: false });

  await db.collection("userNotificationState").updateOne(
    { userId },
    { $set: { unreadCount: actualCount, updatedAt: new Date() } },
    { upsert: true }
  );
}
```

Run this as a scheduled job nightly.

## Summary

Efficient read/unread tracking in MongoDB combines a boolean `isRead` field with a partial index (indexing only unread documents) and a separate counter document for O(1) unread count lookups. Mark individual notifications read with `findOneAndUpdate` to detect actual state changes before decrementing the counter. For bulk mark-as-read, update the counter synchronously and run `updateMany` in the background. Reconcile counters periodically with a scheduled job.
