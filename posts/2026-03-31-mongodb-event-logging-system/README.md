---
title: "How to Model an Event Logging System in MongoDB"
author: "nawazdhandala"
tags: ["MongoDB", "Schema design", "Event", "Logging"]
description: "Learn how to design a MongoDB schema for an event logging system with structured events, TTL-based retention, efficient querying by type and time, and aggregation for analytics."
---

# How to Model an Event Logging System in MongoDB

An event logging system records discrete actions that occur in your application: user logins, page views, purchases, errors, admin actions, and more. MongoDB is well-suited to this use case because its flexible schema accommodates different event shapes, and its aggregation pipeline enables rich analytics on the logged data.

## Event Document Schema

Use a common envelope for all events with a `type` discriminator, and store type-specific data in a `data` subdocument.

```javascript
// User login event
{
  _id: ObjectId("64a1b2c3d4e5f6789abc0001"),
  type: "user.login",
  severity: "info",
  userId: ObjectId("64a1b2c3d4e5f6789abc1001"),
  sessionId: "sess-abc123",
  ipAddress: "192.168.1.1",
  userAgent: "Mozilla/5.0 ...",
  data: {
    authMethod: "password",
    mfaUsed: true
  },
  createdAt: new Date("2024-06-15T10:23:00Z"),
  ttl: new Date("2024-12-15T10:23:00Z")  // For TTL-based expiry
}

// Purchase event
{
  _id: ObjectId("64a1b2c3d4e5f6789abc0002"),
  type: "order.completed",
  severity: "info",
  userId: ObjectId("64a1b2c3d4e5f6789abc1001"),
  sessionId: "sess-abc123",
  data: {
    orderId: ObjectId("64a1b2c3d4e5f6789abc2001"),
    total: 149.99,
    currency: "USD",
    itemCount: 3
  },
  createdAt: new Date("2024-06-15T10:45:00Z"),
  ttl: new Date("2026-06-15T10:45:00Z")  // Longer retention for financial events
}

// Error event
{
  _id: ObjectId("64a1b2c3d4e5f6789abc0003"),
  type: "error.unhandled",
  severity: "error",
  userId: null,
  data: {
    message: "Cannot read properties of undefined",
    stack: "TypeError: ...",
    requestPath: "/api/products/123",
    requestMethod: "GET"
  },
  createdAt: new Date("2024-06-15T10:50:00Z"),
  ttl: new Date("2024-09-15T10:50:00Z")
}
```

## Indexes

```javascript
// Primary query patterns
db.events.createIndex({ createdAt: -1 });
db.events.createIndex({ type: 1, createdAt: -1 });
db.events.createIndex({ userId: 1, createdAt: -1 });
db.events.createIndex({ severity: 1, createdAt: -1 });

// TTL index for automatic cleanup
db.events.createIndex({ ttl: 1 }, { expireAfterSeconds: 0 });
```

The TTL index works by deleting documents where the `ttl` field value is in the past. Setting different `ttl` values per event type lets you implement per-type retention policies.

## Logging Events from Application Code

```javascript
class EventLogger {
  constructor(db, defaultRetentionDays = 90) {
    this.collection = db.collection("events");
    this.defaultRetentionDays = defaultRetentionDays;
  }

  async log(type, data, options = {}) {
    const {
      userId = null,
      sessionId = null,
      severity = "info",
      retentionDays = this.defaultRetentionDays,
      ipAddress = null
    } = options;

    const now = new Date();
    const ttl = new Date(now);
    ttl.setDate(ttl.getDate() + retentionDays);

    await this.collection.insertOne({
      type,
      severity,
      userId,
      sessionId,
      ipAddress,
      data,
      createdAt: now,
      ttl
    });
  }
}

const logger = new EventLogger(db);

// Log a login event (90-day retention)
await logger.log("user.login", { authMethod: "password", mfaUsed: true }, {
  userId: currentUser._id,
  sessionId: req.sessionID,
  severity: "info",
  ipAddress: req.ip
});

// Log an error event (30-day retention)
await logger.log("error.unhandled", { message: err.message, stack: err.stack }, {
  severity: "error",
  retentionDays: 30
});
```

## Event Type Taxonomy

Organize event types using dot-notation namespaces.

```javascript
const EVENT_TYPES = {
  // Auth
  USER_LOGIN: "user.login",
  USER_LOGOUT: "user.logout",
  USER_FAILED_LOGIN: "user.login.failed",
  PASSWORD_CHANGED: "user.password.changed",

  // Orders
  ORDER_CREATED: "order.created",
  ORDER_PAID: "order.paid",
  ORDER_SHIPPED: "order.shipped",
  ORDER_CANCELLED: "order.cancelled",

  // Admin
  ADMIN_USER_BANNED: "admin.user.banned",
  ADMIN_CONTENT_DELETED: "admin.content.deleted",

  // Errors
  ERROR_UNHANDLED: "error.unhandled",
  ERROR_VALIDATION: "error.validation"
};
```

## Querying Events

```javascript
// All events for a user in the last 7 days
const weekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);

const userEvents = await db.collection("events")
  .find({ userId: currentUser._id, createdAt: { $gte: weekAgo } })
  .sort({ createdAt: -1 })
  .limit(100)
  .toArray();

// All error events in the last hour
const hourAgo = new Date(Date.now() - 60 * 60 * 1000);
const errors = await db.collection("events")
  .find({ severity: "error", createdAt: { $gte: hourAgo } })
  .sort({ createdAt: -1 })
  .toArray();
```

## Aggregation for Analytics

```javascript
// Event counts by type for the last 24 hours
const dayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);

const eventSummary = await db.collection("events").aggregate([
  { $match: { createdAt: { $gte: dayAgo } } },
  {
    $group: {
      _id: "$type",
      count: { $sum: 1 },
      latestAt: { $max: "$createdAt" }
    }
  },
  { $sort: { count: -1 } }
]).toArray();
```

```javascript
// Login failures by IP address (rate limiting analysis)
const suspiciousIPs = await db.collection("events").aggregate([
  {
    $match: {
      type: "user.login.failed",
      createdAt: { $gte: new Date(Date.now() - 3600000) }
    }
  },
  {
    $group: {
      _id: "$ipAddress",
      failedAttempts: { $sum: 1 }
    }
  },
  { $match: { failedAttempts: { $gt: 10 } } },
  { $sort: { failedAttempts: -1 } }
]).toArray();
```

## Capped Collections for Fixed-Size Logs

For debug or trace logs where you only need the most recent N entries, use a capped collection.

```javascript
db.createCollection("debugLogs", {
  capped: true,
  size: 104857600,  // 100 MB cap
  max: 1000000      // Max 1 million documents
});
```

Capped collections automatically remove the oldest documents when the cap is reached.

## Summary

Design your event logging collection with a common envelope containing `type`, `severity`, `userId`, `sessionId`, `data`, and `createdAt` fields. Use a dot-notation namespace for event types to enable prefix matching (for example, all `order.*` events). Implement TTL-based retention with per-event `ttl` fields and a TTL index so that different event types expire at different times. Index on `type + createdAt` and `userId + createdAt` for the most common query patterns. Use the aggregation pipeline for event analytics and summaries.
