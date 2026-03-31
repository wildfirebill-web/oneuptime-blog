# How to Build an In-App Notification System with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Notification, Schema, Index, Realtime

Description: Design a scalable MongoDB schema for in-app notifications with unread counts, pagination, category filtering, and efficient per-user queries.

---

## Schema Design

In-app notifications are user-specific and need to support pagination, read/unread filtering, and category-based filtering:

```javascript
db.notifications.insertOne({
  userId: "user_abc123",
  type: "order_shipped", // order_shipped | payment_received | mention | system
  category: "orders",
  title: "Your order has shipped",
  body: "Order #ORD-9876 is on its way. Expected delivery: Thursday.",
  actionUrl: "/orders/ORD-9876",
  imageUrl: "https://cdn.example.com/icons/shipping.png",
  metadata: { orderId: "ORD-9876" },
  isRead: false,
  readAt: null,
  createdAt: new Date(),
  expiresAt: new Date(Date.now() + 90 * 24 * 60 * 60 * 1000), // 90 days
});
```

## Required Indexes

```javascript
// Primary query: fetch recent notifications for a user
db.notifications.createIndex({ userId: 1, createdAt: -1 });

// Filter by unread only
db.notifications.createIndex({ userId: 1, isRead: 1, createdAt: -1 });

// Filter by category
db.notifications.createIndex({ userId: 1, category: 1, createdAt: -1 });

// TTL cleanup after 90 days
db.notifications.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

## Fetching Notifications with Pagination

Use cursor-based pagination (keyset pagination) for consistent results as new notifications arrive:

```javascript
async function getNotifications(userId, lastCreatedAt = null, limit = 20) {
  const query = { userId };
  if (lastCreatedAt) {
    query.createdAt = { $lt: new Date(lastCreatedAt) };
  }

  return db
    .collection("notifications")
    .find(query)
    .sort({ createdAt: -1 })
    .limit(limit)
    .toArray();
}
```

## Unread Count

Keep a separate counter for O(1) unread count lookups instead of counting on every request:

```javascript
// Increment unread count when a notification is created
db.userNotificationState.updateOne(
  { userId: "user_abc123" },
  { $inc: { unreadCount: 1 } },
  { upsert: true }
);

// Decrement when marked as read
db.userNotificationState.updateOne(
  { userId: "user_abc123" },
  { $inc: { unreadCount: -1 } }
);

// Get unread count
const state = await db
  .collection("userNotificationState")
  .findOne({ userId: "user_abc123" }, { projection: { unreadCount: 1 } });
```

## Mark All as Read

```javascript
async function markAllRead(userId) {
  const session = client.startSession();
  await session.withTransaction(async () => {
    await db.collection("notifications").updateMany(
      { userId, isRead: false },
      { $set: { isRead: true, readAt: new Date() } },
      { session }
    );
    await db
      .collection("userNotificationState")
      .updateOne({ userId }, { $set: { unreadCount: 0 } }, { session });
  });
  await session.endSession();
}
```

## Delete Old Notifications

Beyond TTL, allow users to manually clear notifications:

```javascript
// Delete all read notifications for a user
db.notifications.deleteMany({ userId, isRead: true });

// Delete notifications older than 30 days regardless of read state
const cutoff = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
db.notifications.deleteMany({ userId, createdAt: { $lt: cutoff } });
```

## Real-time Delivery with Change Streams

Push new notifications to connected clients using Change Streams:

```javascript
const changeStream = db.collection("notifications").watch([
  { $match: { operationType: "insert", "fullDocument.userId": targetUserId } },
]);

changeStream.on("change", (change) => {
  websocket.send(JSON.stringify(change.fullDocument));
});
```

## Summary

Build an in-app notification system with a `notifications` collection indexed on `userId` and `createdAt` for efficient per-user pagination, plus a `userNotificationState` collection for O(1) unread counts. Use cursor-based pagination for consistent ordering, transactions when marking all as read to keep the counter in sync, TTL indexes for automatic expiry, and Change Streams to push new notifications to connected WebSocket clients.
