---
title: "How to Model a Notification System in MongoDB"
author: "nawazdhandala"
tags: ["MongoDB", "Schema design", "Notification", "Data modeling"]
description: "Learn how to design a MongoDB schema for a notification system with efficient unread counts, inbox pagination, multi-channel delivery, and automatic TTL-based cleanup."
---

# How to Model a Notification System in MongoDB

A notification system delivers messages to users about activity that concerns them: new messages, mentions, replies, order updates, and security alerts. The schema must support fast unread counts, paginated inbox queries, multi-channel delivery (in-app, email, push), and automatic cleanup of old notifications.

## Notification Document Schema

```javascript
db.notifications.insertOne({
  _id: ObjectId("64a1b2c3d4e5f6789abc0001"),
  userId: ObjectId("64a1b2c3d4e5f6789abc1001"),  // Recipient
  type: "comment.reply",
  title: "Alice replied to your comment",
  body: "Great point! I also think that...",
  imageUrl: "https://cdn.example.com/avatars/alice.jpg",
  read: false,
  readAt: null,

  // Action link -- what clicking the notification opens
  action: {
    type: "link",
    url: "/posts/64a1b2c3d4e5f6789abc2001#comment-64a1b2c3d4e5f6789abc3001"
  },

  // Source context -- the actor and the related entity
  actor: {
    userId: ObjectId("64a1b2c3d4e5f6789abc0002"),
    displayName: "Alice",
    avatarUrl: "https://cdn.example.com/avatars/alice.jpg"
  },
  entityType: "comment",
  entityId: ObjectId("64a1b2c3d4e5f6789abc3001"),

  // Delivery channels
  channels: {
    inApp: { delivered: true, deliveredAt: new Date() },
    email: { delivered: false, deliveredAt: null },
    push: { delivered: true, deliveredAt: new Date() }
  },

  createdAt: new Date("2024-06-15T10:23:00Z"),
  expiresAt: new Date("2024-09-15T10:23:00Z")  // TTL field
});
```

## Indexes

```javascript
// Primary inbox query: unread first, then by time
db.notifications.createIndex({ userId: 1, read: 1, createdAt: -1 });

// Read status queries
db.notifications.createIndex({ userId: 1, createdAt: -1 });

// Entity-based lookup (for deduplication)
db.notifications.createIndex({ userId: 1, type: 1, entityId: 1 });

// TTL auto-cleanup
db.notifications.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);
```

## Getting the Unread Count

```javascript
async function getUnreadCount(db, userId) {
  return db.collection("notifications").countDocuments({
    userId,
    read: false
  });
}
```

For very high-traffic applications, maintaining a denormalized `unreadCount` on the user document can be faster than a `countDocuments` on every request.

```javascript
// Increment unread count on the user when a notification is created
await db.collection("users").updateOne(
  { _id: recipientId },
  { $inc: { unreadNotificationCount: 1 } }
);

// Decrement when notifications are marked as read
await db.collection("users").updateOne(
  { _id: userId },
  { $inc: { unreadNotificationCount: -markedCount } }
);
```

## Loading the Inbox

```javascript
async function getNotifications(db, userId, options = {}) {
  const { limit = 20, before = null, unreadOnly = false } = options;

  const filter = { userId };
  if (unreadOnly) filter.read = false;
  if (before) filter.createdAt = { $lt: before };

  return db.collection("notifications")
    .find(filter)
    .sort({ createdAt: -1 })
    .limit(limit)
    .toArray();
}
```

## Marking Notifications as Read

```javascript
// Mark a single notification as read
async function markAsRead(db, userId, notificationId) {
  const result = await db.collection("notifications").updateOne(
    { _id: notificationId, userId, read: false },
    { $set: { read: true, readAt: new Date() } }
  );
  if (result.modifiedCount > 0) {
    await db.collection("users").updateOne(
      { _id: userId },
      { $inc: { unreadNotificationCount: -1 } }
    );
  }
}

// Mark all notifications as read
async function markAllAsRead(db, userId) {
  const result = await db.collection("notifications").updateMany(
    { userId, read: false },
    { $set: { read: true, readAt: new Date() } }
  );
  await db.collection("users").updateOne(
    { _id: userId },
    { $set: { unreadNotificationCount: 0 } }
  );
  return result.modifiedCount;
}
```

## Creating Notifications

```javascript
async function createNotification(db, options) {
  const {
    userId,
    type,
    title,
    body,
    actor,
    entityType,
    entityId,
    actionUrl,
    channels = ["inApp"],
    retentionDays = 90
  } = options;

  const now = new Date();
  const expiresAt = new Date(now);
  expiresAt.setDate(expiresAt.getDate() + retentionDays);

  const channelStatus = {};
  channels.forEach((ch) => {
    channelStatus[ch] = { delivered: false, deliveredAt: null };
  });

  const notification = {
    _id: ObjectId(),
    userId,
    type,
    title,
    body,
    read: false,
    readAt: null,
    action: { type: "link", url: actionUrl },
    actor,
    entityType,
    entityId,
    channels: channelStatus,
    createdAt: now,
    expiresAt
  };

  await db.collection("notifications").insertOne(notification);
  await db.collection("users").updateOne(
    { _id: userId },
    { $inc: { unreadNotificationCount: 1 } }
  );

  return notification;
}
```

## Notification Type Taxonomy

```javascript
const NOTIFICATION_TYPES = {
  COMMENT_REPLY: "comment.reply",
  POST_MENTION: "post.mention",
  COMMENT_LIKE: "comment.like",
  FOLLOW_NEW: "follow.new",
  ORDER_SHIPPED: "order.shipped",
  ORDER_DELIVERED: "order.delivered",
  PAYMENT_FAILED: "payment.failed",
  SECURITY_LOGIN_NEW_DEVICE: "security.login.new_device",
  SECURITY_PASSWORD_CHANGED: "security.password.changed"
};
```

## Deduplication: Preventing Notification Floods

For high-frequency events (many likes on the same post), use upsert with `$setOnInsert` to collapse duplicates into a single notification.

```javascript
async function createOrUpdateLikeNotification(db, postOwnerId, likerId, postId) {
  await db.collection("notifications").updateOne(
    {
      userId: postOwnerId,
      type: "post.like",
      entityId: postId,
      read: false
    },
    {
      $setOnInsert: {
        _id: ObjectId(),
        userId: postOwnerId,
        type: "post.like",
        entityId: postId,
        entityType: "post",
        read: false,
        createdAt: new Date(),
        expiresAt: new Date(Date.now() + 90 * 86400000)
      },
      $set: {
        title: "Someone liked your post",
        updatedAt: new Date()
      },
      $addToSet: { "data.likerIds": likerId },
      $inc: { "data.likeCount": 1 }
    },
    { upsert: true }
  );
}
```

## Summary

A MongoDB notification schema centers on a `notifications` collection with `userId`, `type`, `read`, `createdAt`, and optional `expiresAt` fields. Index on `userId + read + createdAt` for efficient inbox and unread count queries. Use a TTL index on `expiresAt` for automatic cleanup. Maintain a denormalized `unreadNotificationCount` on the user document for O(1) badge count reads. Use upsert with `$addToSet` for deduplication of high-frequency events. Organize notification types with dot-notation namespaces to make filtering by category straightforward.
