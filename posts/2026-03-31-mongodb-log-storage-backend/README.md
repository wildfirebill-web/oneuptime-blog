# How to Use MongoDB as a Log Storage Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Logging, Log Storage, Backend, Operations

Description: Learn how to use MongoDB as a scalable log storage backend with structured log documents, indexing strategies, and TTL-based automatic expiry.

---

## Overview

MongoDB is a strong fit for log storage because logs are naturally document-shaped: variable fields, nested context objects, and append-only write patterns. Its flexible schema accommodates different log formats, while TTL indexes handle automatic expiry without a separate cleanup job.

## Structuring Log Documents

Design log documents with consistent top-level fields and flexible metadata:

```javascript
{
  "_id": ObjectId("..."),
  "timestamp": ISODate("2026-03-31T10:00:00Z"),
  "level": "error",
  "service": "payment-api",
  "message": "Payment gateway timeout",
  "context": {
    "orderId": "ord-123",
    "userId": "user-456",
    "durationMs": 5003
  },
  "host": "payment-api-pod-7f9b",
  "traceId": "abc123def456"
}
```

Using `ISODate` for `timestamp` enables range queries and TTL indexes. The `context` object can vary between log entries without schema changes.

## Creating the Logs Collection

```javascript
db.createCollection("app_logs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["timestamp", "level", "service", "message"],
      properties: {
        timestamp: { bsonType: "date" },
        level: { enum: ["debug", "info", "warn", "error", "fatal"] },
        service: { bsonType: "string" },
        message: { bsonType: "string" }
      }
    }
  }
})
```

Schema validation ensures consistent required fields while still allowing arbitrary `context` fields.

## Indexing for Common Query Patterns

```javascript
// Query logs by service and time range
db.app_logs.createIndex({ service: 1, timestamp: -1 })

// Query by log level and time
db.app_logs.createIndex({ level: 1, timestamp: -1 })

// Full-text search on messages
db.app_logs.createIndex({ message: "text" })
```

## Adding TTL for Automatic Expiry

```javascript
// Automatically delete logs older than 30 days
db.app_logs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 2592000 }
)
```

The TTL background thread runs every 60 seconds and deletes expired documents automatically.

## Writing Logs from a Node.js Application

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient(process.env.MONGODB_URI);
const db = client.db('logs');
const collection = db.collection('app_logs');

async function writeLog(level, service, message, context = {}) {
  await collection.insertOne({
    timestamp: new Date(),
    level,
    service,
    message,
    context,
    host: process.env.HOSTNAME,
  });
}

// Usage
await writeLog('error', 'payment-api', 'Gateway timeout', { orderId: 'ord-123' });
```

## Querying Logs

```javascript
// Last 100 errors from the payment service in the past hour
const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);
const errors = await collection.find({
  service: 'payment-api',
  level: 'error',
  timestamp: { $gte: oneHourAgo }
}).sort({ timestamp: -1 }).limit(100).toArray();
```

## Best Practices

- Use bulk inserts (`insertMany`) when logging high-frequency events to reduce round trips.
- Add a `traceId` field to correlate logs across microservices.
- Enable WiredTiger compression on the logs collection to reduce storage by 50-70%.
- For very high write volumes (millions of logs/day), consider capped collections or Atlas Online Archive for cost efficiency.

## Summary

MongoDB makes an effective log storage backend due to its flexible document model, powerful indexing, and TTL-based expiry. Structure logs with consistent required fields, index for your query patterns, and use TTL indexes to manage retention automatically.
