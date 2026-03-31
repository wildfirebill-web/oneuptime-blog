# How to Use TTL Indexes for Automatic Document Expiration in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Ttl Indexes, Document Expiration, Index Management, Database Maintenance

Description: TTL indexes in MongoDB automatically delete documents after a specified time, making them ideal for session data, logs, and temporary records.

---

## What Are TTL Indexes

A TTL (Time-To-Live) index is a special single-field index in MongoDB that periodically removes documents once they reach a certain age. MongoDB runs a background thread every 60 seconds that scans the collection and deletes expired documents.

TTL indexes are commonly used for:
- Session tokens and authentication records
- Application logs and event streams
- Cache entries and temporary data
- OTP codes and verification tokens
- Scheduled task records

## Creating a Basic TTL Index

To create a TTL index, use `createIndex` with the `expireAfterSeconds` option on a `Date` field.

```javascript
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }
)
```

This removes any document in the `sessions` collection where `createdAt` is older than 1 hour (3600 seconds).

The field referenced by the TTL index must be a BSON date type or an array of BSON dates. If the field holds an array, MongoDB uses the earliest date in the array.

## Inserting Documents with Expiration

```javascript
db.sessions.insertOne({
  userId: "user_abc123",
  token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  createdAt: new Date()
})
```

The document will be automatically removed approximately 1 hour after `createdAt`.

## Setting Expiration on a Specific Date Field

You can also use `expireAfterSeconds: 0` and set a specific expiration timestamp per document:

```javascript
db.notifications.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
)

db.notifications.insertOne({
  message: "Your subscription renews tomorrow.",
  userId: "user_xyz",
  expiresAt: new Date("2026-04-01T00:00:00Z")
})
```

With `expireAfterSeconds: 0`, MongoDB deletes each document at the exact datetime stored in `expiresAt`. This gives you per-document control over expiration.

## Viewing Existing TTL Indexes

Use `getIndexes()` to inspect indexes on a collection and find TTL configurations:

```javascript
db.sessions.getIndexes()
```

Example output:

```json
[
  {
    "v": 2,
    "key": { "_id": 1 },
    "name": "_id_"
  },
  {
    "v": 2,
    "key": { "createdAt": 1 },
    "name": "createdAt_1",
    "expireAfterSeconds": 3600
  }
]
```

## Modifying the TTL Value

You cannot drop and recreate a TTL index easily on large collections without downtime. Use `collMod` to update `expireAfterSeconds` on an existing TTL index:

```javascript
db.runCommand({
  collMod: "sessions",
  index: {
    name: "createdAt_1",
    expireAfterSeconds: 7200
  }
})
```

This updates the TTL to 2 hours without rebuilding the index.

## Understanding the Deletion Frequency

MongoDB's TTL monitor thread runs approximately every 60 seconds, so there can be a delay of up to 60 seconds between when a document technically expires and when it is actually deleted. During replica set failover or heavy load, the delay can be longer.

Do not rely on TTL for security-critical exact-time deletion. Instead, use application-level checks and treat TTL as an eventual cleanup mechanism.

## TTL Index Limitations

TTL indexes have several restrictions:

- The indexed field must be a Date type (not a string or number)
- TTL indexes cannot be compound - they must be single-field indexes
- Capped collections do not support TTL indexes
- TTL indexes cannot be created on the `_id` field
- You cannot convert a regular index to a TTL index

```javascript
// WRONG - this will fail because the field is a string not a Date
db.logs.insertOne({ expiry: "2026-04-01" })  // string - TTL won't work

// CORRECT - use a Date object
db.logs.insertOne({ expiry: new Date("2026-04-01") })
```

## Practical Example - OTP Expiry

Here is a complete pattern for one-time password tokens that expire in 10 minutes:

```javascript
// Create collection and TTL index
db.otpTokens.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 600 }
)

// Insert OTP record
db.otpTokens.insertOne({
  userId: "user_001",
  code: "482910",
  createdAt: new Date(),
  used: false
})

// Verify OTP (check existence - expired docs are already gone)
const otp = db.otpTokens.findOne({
  userId: "user_001",
  code: "482910",
  used: false
})

if (otp) {
  db.otpTokens.updateOne(
    { _id: otp._id },
    { $set: { used: true } }
  )
  console.log("OTP verified successfully")
} else {
  console.log("OTP expired or invalid")
}
```

## Monitoring TTL Index Activity

Check whether the TTL background job is running and how many documents it has deleted using the serverStatus command:

```javascript
db.serverStatus().metrics.ttl
```

Example output:

```json
{
  "deletedDocuments": 14823,
  "passes": 302
}
```

`passes` is the number of times the TTL monitor has run, and `deletedDocuments` is the cumulative count of removed documents since the server started.

## Summary

TTL indexes are a lightweight, built-in MongoDB mechanism for automatically expiring documents based on a date field. You can apply a fixed expiration window using `expireAfterSeconds` or control per-document expiration by using `expireAfterSeconds: 0` with a specific date field. Remember that deletion is not instantaneous - the TTL monitor runs every 60 seconds - so design applications to tolerate brief delays in document removal.
