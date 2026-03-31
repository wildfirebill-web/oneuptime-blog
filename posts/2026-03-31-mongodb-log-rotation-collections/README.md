# How to Set Up Log Rotation for MongoDB Collections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Logging, Log Rotation, TTL, Operation

Description: Learn how to implement log rotation for MongoDB log collections using TTL indexes, date-partitioned collections, and archival scripts to manage storage growth.

---

## Overview

Log rotation in MongoDB means automatically removing old log documents and optionally archiving them. Unlike file-based log rotation, MongoDB uses TTL indexes for automatic expiry or date-partitioned collections for manual archival workflows.

## Approach 1 - TTL Index for Automatic Expiry

The simplest approach: create a TTL index on the `timestamp` field. MongoDB's background TTL thread deletes documents after the specified duration.

```javascript
// Expire logs after 30 days
db.app_logs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 2592000 }
)
```

To change the TTL on an existing index:

```javascript
db.runCommand({
  collMod: "app_logs",
  index: {
    keyPattern: { timestamp: 1 },
    expireAfterSeconds: 604800  // Change to 7 days
  }
})
```

Verify the TTL setting:

```javascript
db.app_logs.getIndexes().filter(i => i.expireAfterSeconds)
```

## Approach 2 - Date-Partitioned Collections

For longer retention with tiered storage, use separate collections per month:

```javascript
function getLogCollection(db, date) {
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  return db.collection(`app_logs_${year}_${month}`);
}

// Write to the current month's collection
const collection = getLogCollection(db, new Date());
await collection.insertOne({ timestamp: new Date(), level: "info", message: "Started" });
```

At the start of each month, create the new collection and set up its indexes:

```javascript
async function initMonthCollection(db, year, month) {
  const name = `app_logs_${year}_${String(month).padStart(2, '0')}`;
  await db.createCollection(name);
  await db.collection(name).createIndex({ level: 1, timestamp: -1 });
  await db.collection(name).createIndex({ service: 1, timestamp: -1 });
}
```

To rotate (drop) collections older than 3 months:

```javascript
async function rotateOldCollections(db) {
  const cutoff = new Date();
  cutoff.setMonth(cutoff.getMonth() - 3);
  const cutoffName = `app_logs_${cutoff.getFullYear()}_${String(cutoff.getMonth() + 1).padStart(2, '0')}`;

  const collections = await db.listCollections({ name: /^app_logs_/ }).toArray();
  for (const col of collections) {
    if (col.name < cutoffName) {
      console.log(`Dropping old log collection: ${col.name}`);
      await db.collection(col.name).drop();
    }
  }
}
```

## Approach 3 - Archive Before Delete

Before deleting old logs, archive them to Atlas Online Archive or a cold storage bucket:

```bash
# Export a month's logs to JSONL format
mongoexport \
  --uri="$MONGODB_URI" \
  --collection="app_logs_2025_12" \
  --out="/archive/app_logs_2025_12.jsonl" \
  --type=json

# Compress the archive
gzip /archive/app_logs_2025_12.jsonl
```

After confirming the archive is complete, drop the collection:

```javascript
db.app_logs_2025_12.drop()
```

## Scheduling Rotation

Run rotation scripts via cron:

```bash
# Run log rotation every Sunday at 2 AM
0 2 * * 0 /usr/local/bin/node /opt/scripts/rotate-logs.js >> /var/log/mongo-rotation.log 2>&1
```

## Best Practices

- For simple retention requirements under 90 days, use TTL indexes - they require no maintenance.
- For compliance requirements where you need to audit log history, use date-partitioned collections with archival to object storage before dropping.
- Monitor the size of your log collection with `db.app_logs.stats().storageSize` and alert when it grows unexpectedly.

## Summary

MongoDB log rotation uses TTL indexes for automatic expiry, date-partitioned collections for tiered storage, and `mongoexport` plus collection drops for archive-then-delete workflows. Choose the approach that matches your retention requirements and compliance obligations.
