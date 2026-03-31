# How to Create a TTL Index to Auto-Expire Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, TTL Index, Auto-Expire, Data Lifecycle

Description: Learn how to create TTL indexes in MongoDB to automatically delete documents after a specified time period without application-level cleanup logic.

---

## Overview

A TTL (Time To Live) index is a special MongoDB index that automatically deletes documents from a collection after a specified number of seconds have passed since the value of the indexed date field. TTL indexes are useful for managing session data, log entries, temporary tokens, cache documents, and any data with a defined expiry.

## Creating a TTL Index

```javascript
// Delete documents 3600 seconds (1 hour) after "createdAt"
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })

// Delete documents 86400 seconds (24 hours) after "createdAt"
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })

// Delete documents at a specific expiry date stored in the document
// Set expireAfterSeconds to 0 and store the expiry date in the field
db.tokens.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })
```

## How TTL Indexes Work

```text
- MongoDB runs a background task (TTLMonitor) every 60 seconds
- It scans TTL indexes and deletes documents where:
    (indexedDateField + expireAfterSeconds) <= currentTime
- Documents are not expired at exactly the right second - there may be up to
  60 seconds of delay plus time to complete the deletion
- Deletion is performed as a low-priority background operation
```

## Practical Example - Session Management

```javascript
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 1800 })  // 30 minutes

// Insert a session - it will be deleted automatically after 30 minutes
db.sessions.insertOne({
  sessionId: "abc123",
  userId: "user456",
  createdAt: new Date(),
  data: { cart: ["item1", "item2"] }
})
```

## Using expiresAt Field (expireAfterSeconds: 0)

When `expireAfterSeconds` is 0, MongoDB deletes documents at the date/time stored in the indexed field:

```javascript
db.tokens.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })

// Token expires in 7 days
db.tokens.insertOne({
  token: "xyz789",
  userId: "user123",
  type: "passwordReset",
  expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
})

// Token with custom expiry
db.tokens.insertOne({
  token: "abc456",
  userId: "user789",
  type: "emailVerification",
  expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000)  // 24 hours
})
```

## Modifying the TTL Expiry Value

You cannot drop and recreate a TTL index easily in production. Use `collMod` to change the TTL value:

```javascript
db.runCommand({
  collMod: "sessions",
  index: {
    keyPattern: { createdAt: 1 },
    expireAfterSeconds: 7200   // Change from 1 hour to 2 hours
  }
})
```

## TTL Index Limitations

```text
- The indexed field must be a Date type (or an array of Dates - uses earliest)
- TTL indexes must be single field (not compound)
- Cannot create TTL index on _id field
- Capped collections do not support TTL indexes
- TTL does not work on replica set secondaries (only primary performs deletion)
- Documents without the indexed date field (or with null) are not expired
- TTL Monitor runs every 60 seconds - expect up to 60+ seconds of delay
```

## TTL Index with Arrays of Dates

If the TTL field contains an array of dates, MongoDB uses the lowest (earliest) date in the array:

```javascript
db.events.createIndex({ scheduledAt: 1 }, { expireAfterSeconds: 0 })

// Document expires at the earliest date in the array
db.events.insertOne({
  name: "Recurring task",
  scheduledAt: [
    new Date("2024-06-01"),
    new Date("2024-05-15"),  // This is earliest - document expires here
    new Date("2024-07-01")
  ]
})
```

## Monitoring TTL Deletions

```javascript
// Check TTL monitor status in server status
db.adminCommand({ serverStatus: 1 }).metrics.ttl
// Returns: { deletedDocuments: N, passes: N }

// Verify documents are being expired
db.sessions.countDocuments()
// Count should decrease over time as TTL expires old documents
```

## Common Use Cases

```javascript
// 1. Web session storage (30 minutes)
db.sessions.createIndex({ lastActivity: 1 }, { expireAfterSeconds: 1800 })

// 2. Email verification tokens (24 hours)
db.verificationTokens.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })

// 3. Rate limiting records (1 minute window)
db.rateLimits.createIndex({ windowStart: 1 }, { expireAfterSeconds: 60 })

// 4. API response cache (5 minutes)
db.responseCache.createIndex({ cachedAt: 1 }, { expireAfterSeconds: 300 })

// 5. Audit logs retained for 90 days
db.auditLogs.createIndex({ loggedAt: 1 }, { expireAfterSeconds: 7776000 })
```

## Summary

TTL indexes automatically delete documents after a specified number of seconds from the value of a date field, eliminating the need for application-level cleanup jobs. Set `expireAfterSeconds` to a positive integer to expire documents relative to a field value, or 0 to expire at a specific date stored in the document. TTL deletion runs every 60 seconds with possible additional delay. Use `collMod` to adjust the expiry without recreating the index in production.
