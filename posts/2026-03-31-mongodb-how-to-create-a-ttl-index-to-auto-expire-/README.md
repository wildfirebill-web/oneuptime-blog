# How to Create a TTL Index to Auto-Expire Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, TTL Index, Data Expiration, Database

Description: Learn how to create TTL (Time-To-Live) indexes in MongoDB to automatically delete documents after a specified time, ideal for sessions, logs, and temporary data.

---

## Overview

A TTL (Time-To-Live) index in MongoDB is a special single-field index on a date-type field that automatically removes documents after a specified number of seconds. MongoDB runs a background task every 60 seconds to delete expired documents, making TTL indexes ideal for managing temporary data like user sessions, audit logs, cache entries, and time-bound notifications.

## Creating a TTL Index

Use `createIndex()` with the `expireAfterSeconds` option on a date field:

```javascript
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // expire after 1 hour (3600 seconds)
)
```

Documents where `createdAt` is older than the specified number of seconds are automatically deleted.

## How TTL Deletion Works

```text
1. MongoDB runs a TTL monitor thread every 60 seconds
2. It finds documents where: current time >= indexedDateField + expireAfterSeconds
3. Documents matching the condition are deleted
4. Deletion is not immediate - there is up to a 60-second delay
5. TTL deletion uses the same delete operation as a manual deleteMany()
```

## Common TTL Use Cases

Session storage that expires after 30 minutes:

```javascript
db.userSessions.createIndex(
  { lastAccessedAt: 1 },
  { expireAfterSeconds: 1800 }
)

// Insert a session - it expires 30 minutes after lastAccessedAt
db.userSessions.insertOne({
  userId: "user_123",
  token: "abc...",
  lastAccessedAt: new Date()
})

// Update lastAccessedAt on each access to "refresh" the session
db.userSessions.updateOne(
  { token: "abc..." },
  { $set: { lastAccessedAt: new Date() } }
)
```

Log retention for 90 days:

```javascript
db.auditLogs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 7776000 }  // 90 days
)
```

## Expiring at a Specific Time (expireAfterSeconds: 0)

Set `expireAfterSeconds: 0` and store the exact expiry time in the indexed field. Documents expire at the time stored in the field:

```javascript
db.notifications.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
)

// Set explicit expiration time per document
db.notifications.insertOne({
  userId: "user_123",
  message: "Your free trial ends soon",
  expiresAt: new Date("2025-08-01T00:00:00Z")
})
```

## Requirements and Limitations

```text
Requirements:
- The indexed field must be a BSON Date type or an array of Date values
- Only single-field indexes qualify as TTL indexes
- TTL indexes cannot be compound indexes
- The TTL index must be on a non-array field (or an array of dates)

Limitations:
- Deletion is not instant - up to 60 seconds delay
- TTL monitor can fall behind under heavy load
- Cannot use TTL on the _id field
- Capped collections do not support TTL indexes
```

## Modifying the TTL Value

Change the expiration time on an existing TTL index using `collMod`:

```javascript
db.runCommand({
  collMod: "sessions",
  index: {
    name: "createdAt_1",
    expireAfterSeconds: 7200  // Change to 2 hours
  }
})
```

## Checking TTL Index Status

View TTL index details:

```javascript
db.sessions.getIndexes()
// Look for the "expireAfterSeconds" field in the index definition
```

Monitor TTL deletion activity in server logs (set log verbosity to show TTL operations):

```javascript
db.setLogLevel(1, "index")
```

## Partial TTL Index

Combine TTL with `partialFilterExpression` to expire only documents matching a condition:

```javascript
db.tasks.createIndex(
  { completedAt: 1 },
  {
    expireAfterSeconds: 2592000,  // 30 days
    partialFilterExpression: { status: "completed" }
  }
)
```

This only expires completed tasks - incomplete tasks are never automatically deleted.

## Summary

TTL indexes automate document lifecycle management in MongoDB by deleting expired documents based on a date field and a configurable expiry duration. They are the standard approach for session management, log rotation, notification expiry, and any data with a defined retention window. Use `expireAfterSeconds: 0` when each document needs its own expiry time, and combine with `partialFilterExpression` for conditional expiry. Always account for the up-to-60-second deletion delay when designing TTL-dependent features.
