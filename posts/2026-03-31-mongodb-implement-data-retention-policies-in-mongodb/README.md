# How to Implement Data Retention Policies in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Retention, TTL

Description: Implement MongoDB data retention policies using TTL indexes, scheduled archival pipelines, and collection-level expiry for compliance and storage management.

---

## Why Data Retention Policies Matter

Retaining data longer than necessary violates GDPR's data minimization principle and inflates storage costs. Conversely, deleting data too early may violate legal hold requirements. A formal retention policy defines how long each data type is kept, when it is archived, and when it is permanently deleted.

## Method 1: TTL Indexes for Automatic Expiry

TTL (Time To Live) indexes are the simplest retention mechanism. MongoDB automatically deletes documents after a specified period:

```javascript
// Automatically delete sessions after 30 days of inactivity
db.sessions.createIndex(
  { lastActivityAt: 1 },
  { expireAfterSeconds: 30 * 24 * 60 * 60 }  // 30 days
)

// Delete verification tokens after 24 hours
db.verification_tokens.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 24 * 60 * 60 }
)

// Delete temporary uploads after 7 days
db.temp_uploads.createIndex(
  { uploadedAt: 1 },
  { expireAfterSeconds: 7 * 24 * 60 * 60 }
)
```

For documents with a specific expiry time:

```javascript
// Set a specific expiry timestamp on the document
db.password_reset_tokens.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }  // expire at the exact expiresAt value
)

await db.collection('password_reset_tokens').insertOne({
  token: crypto.randomBytes(32).toString('hex'),
  userId: userId,
  expiresAt: new Date(Date.now() + 60 * 60 * 1000),  // 1 hour from now
})
```

## Method 2: Scheduled Archival for Compliance Data

For data that must be retained but moved out of the primary database:

```javascript
// archive_old_orders.js - Run as a scheduled job (cron/K8s CronJob)
const ARCHIVE_AFTER_DAYS = 365  // 1 year
const DELETE_AFTER_DAYS  = 2555 // 7 years (legal hold)

async function archiveOldOrders() {
  const archiveCutoff = new Date(Date.now() - ARCHIVE_AFTER_DAYS * 86400000)
  const deleteCutoff  = new Date(Date.now() - DELETE_AFTER_DAYS  * 86400000)

  // Move to archive collection
  const toArchive = await db.collection('orders').find({
    createdAt: { $lt: archiveCutoff },
    archived: { $ne: true },
  }).limit(1000).toArray()

  if (toArchive.length > 0) {
    await db.collection('orders_archive').insertMany(
      toArchive.map(o => ({ ...o, archivedAt: new Date() }))
    )
    const ids = toArchive.map(o => o._id)
    await db.collection('orders').deleteMany({ _id: { $in: ids } })
    console.log(`Archived ${toArchive.length} orders`)
  }

  // Permanently delete from archive after legal hold period
  const deleted = await db.collection('orders_archive').deleteMany({
    createdAt: { $lt: deleteCutoff },
  })
  console.log(`Permanently deleted ${deleted.deletedCount} archived orders`)
}
```

## Method 3: Retention Policy Metadata

Track retention policy per collection in a central configuration:

```javascript
await db.collection('retention_policies').insertMany([
  { collection: 'sessions',           retainDays: 30,   method: 'ttl',      legalBasis: 'none' },
  { collection: 'audit_logs',         retainDays: 365,  method: 'archive',  legalBasis: 'compliance' },
  { collection: 'orders',             retainDays: 2555, method: 'archive',  legalBasis: 'financial' },
  { collection: 'marketing_events',   retainDays: 180,  method: 'delete',   legalBasis: 'consent' },
  { collection: 'verification_tokens', retainDays: 1,   method: 'ttl',      legalBasis: 'none' },
])
```

## Monitoring Retention Job Execution

```javascript
async function logRetentionRun(collection, archived, deleted, durationMs) {
  await db.collection('retention_job_log').insertOne({
    runAt: new Date(),
    collection,
    archived,
    deleted,
    durationMs,
    success: true,
  })
}
```

Set up an alert if the retention job has not run in 25 hours:

```javascript
const lastRun = await db.collection('retention_job_log')
  .findOne({ collection: 'orders' }, { sort: { runAt: -1 } })

if (!lastRun || Date.now() - lastRun.runAt > 25 * 3600 * 1000) {
  alerting.send('Retention job has not run for 25+ hours')
}
```

## Summary

MongoDB data retention is implemented through three complementary methods: TTL indexes for short-lived ephemeral data (sessions, tokens), scheduled archival jobs for compliance-sensitive data requiring defined retention periods, and permanent deletion after legal hold windows expire. Store retention policy metadata centrally so compliance auditors can verify policies match implementation, and monitor retention job execution to detect failures before they create compliance gaps.
