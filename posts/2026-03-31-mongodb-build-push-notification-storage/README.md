# How to Build Push Notification Storage with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Notification, Push, Schema, Index

Description: Design a MongoDB schema and indexing strategy for storing push notifications with device tokens, delivery status, and expiry-based cleanup using TTL indexes.

---

## Schema Design

A push notification storage system needs to track the notification payload, the target devices, delivery status, and expiry. Here is a practical schema:

```javascript
db.pushNotifications.insertOne({
  userId: "user_abc123",
  title: "Your order has shipped",
  body: "Your order #ORD-9876 is on its way.",
  data: { orderId: "ORD-9876", screen: "order-detail" },
  devices: [
    { token: "ExponentPushToken[xxx]", platform: "ios", status: "sent" },
    { token: "APA91bH...", platform: "android", status: "pending" },
  ],
  status: "partial", // pending | sent | partial | failed
  createdAt: new Date(),
  sentAt: null,
  expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days
});
```

## Indexes

```javascript
// Query notifications for a specific user, sorted by most recent
db.pushNotifications.createIndex({ userId: 1, createdAt: -1 });

// Query pending notifications for retry jobs
db.pushNotifications.createIndex({ status: 1, createdAt: 1 });

// TTL index - MongoDB auto-deletes expired notifications
db.pushNotifications.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

## Storing Device Tokens per User

Keep a separate collection for device token management:

```javascript
db.deviceTokens.insertOne({
  userId: "user_abc123",
  token: "ExponentPushToken[xxx]",
  platform: "ios",
  appVersion: "3.1.0",
  registeredAt: new Date(),
  lastSeenAt: new Date(),
  active: true,
});

db.deviceTokens.createIndex({ userId: 1, platform: 1 });
db.deviceTokens.createIndex({ token: 1 }, { unique: true });
```

## Sending a Push Notification

When your backend sends a push, look up active tokens and create the notification document:

```javascript
async function sendPushNotification(userId, title, body, data) {
  const tokens = await db
    .collection("deviceTokens")
    .find({ userId, active: true })
    .toArray();

  const devices = tokens.map((t) => ({
    token: t.token,
    platform: t.platform,
    status: "pending",
  }));

  const notification = await db.collection("pushNotifications").insertOne({
    userId,
    title,
    body,
    data,
    devices,
    status: "pending",
    createdAt: new Date(),
    expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
  });

  return notification.insertedId;
}
```

## Updating Delivery Status

After the push provider confirms delivery, update each device's status:

```javascript
// Update a single device's delivery status within the notification
db.pushNotifications.updateOne(
  { _id: notificationId, "devices.token": "ExponentPushToken[xxx]" },
  {
    $set: {
      "devices.$.status": "delivered",
      sentAt: new Date(),
    },
  }
);

// Mark notification as fully sent when all devices are delivered
db.pushNotifications.updateOne(
  {
    _id: notificationId,
    "devices.status": { $not: { $in: ["pending", "failed"] } },
  },
  { $set: { status: "sent" } }
);
```

## Handling Invalid Tokens

When a push provider returns an invalid token error, deactivate it:

```javascript
db.deviceTokens.updateOne(
  { token: invalidToken },
  { $set: { active: false, deactivatedAt: new Date() } }
);
```

## Summary

Build push notification storage in MongoDB with two collections: one for device tokens (with a unique index on token) and one for notification documents with embedded per-device delivery status. Use a TTL index on `expiresAt` for automatic cleanup of old notifications. Index on `userId` and `status` for user history queries and retry job polling. Update device-level status using the positional `$` operator.
