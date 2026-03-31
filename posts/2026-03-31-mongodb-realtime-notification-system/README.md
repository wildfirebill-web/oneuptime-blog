# How to Build a Real-Time Notification System with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Notification, Real-Time, Change Stream, WebSocket

Description: Learn how to build a scalable notification system using MongoDB for notification storage, change streams for real-time delivery, and WebSockets to push alerts to users.

---

## Overview

A notification system needs to persist notifications, deliver them in real time when users are connected, and let users retrieve missed notifications when they reconnect. MongoDB change streams combined with WebSockets provide an elegant solution.

## Schema Design

```javascript
// notifications collection
{
  _id: ObjectId(),
  userId: "user-123",
  type: "order_shipped",
  title: "Your order has shipped",
  body: "Order #ORD-456 is on its way and will arrive by Friday.",
  data: { orderId: "ORD-456", trackingNumber: "1Z999AA10123456784" },
  read: false,
  createdAt: new Date()
}
```

Create indexes for efficient user notification queries:

```javascript
db.notifications.createIndex({ userId: 1, createdAt: -1 })
db.notifications.createIndex({ userId: 1, read: 1 })
```

## Inserting Notifications

```javascript
async function createNotification(db, userId, type, title, body, data = {}) {
  const notification = {
    userId,
    type,
    title,
    body,
    data,
    read: false,
    createdAt: new Date()
  };
  await db.collection("notifications").insertOne(notification);
  return notification;
}
```

## Real-Time Delivery with Change Streams

Watch the notifications collection and push new notifications to connected WebSocket clients:

```javascript
const WebSocket = require("ws");
const { MongoClient } = require("mongodb");

const wss = new WebSocket.Server({ port: 8080 });
const userConnections = new Map();  // userId -> WebSocket

wss.on("connection", (ws, req) => {
  const userId = new URLSearchParams(req.url.slice(1)).get("userId");
  userConnections.set(userId, ws);

  ws.on("close", () => userConnections.delete(userId));
});

async function startNotificationStream(db) {
  const changeStream = db.collection("notifications").watch(
    [{ $match: { operationType: "insert" } }]
  );

  for await (const change of changeStream) {
    const notification = change.fullDocument;
    const ws = userConnections.get(notification.userId);

    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        type: "notification",
        payload: notification
      }));
    }
  }
}
```

## Retrieving Unread Notifications

```javascript
app.get("/api/notifications/:userId", async (req, res) => {
  const { userId } = req.params;
  const notifications = await db.collection("notifications")
    .find({ userId, read: false })
    .sort({ createdAt: -1 })
    .limit(50)
    .toArray();
  res.json(notifications);
});
```

## Marking Notifications as Read

```javascript
app.post("/api/notifications/read", async (req, res) => {
  const { userId, notificationIds } = req.body;
  const ids = notificationIds.map(id => new ObjectId(id));

  await db.collection("notifications").updateMany(
    { _id: { $in: ids }, userId },
    { $set: { read: true } }
  );

  res.json({ success: true });
});
```

## Getting Unread Count Badge

```javascript
app.get("/api/notifications/:userId/count", async (req, res) => {
  const count = await db.collection("notifications")
    .countDocuments({ userId: req.params.userId, read: false });
  res.json({ count });
});
```

## Summary

Build a notification system on MongoDB by storing notifications as documents, watching the collection with a change stream for real-time delivery, and maintaining a WebSocket map of connected users to push alerts instantly. Fall back to the REST API for users who were offline to retrieve missed notifications when they reconnect.
