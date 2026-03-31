# How to Implement Document Archival Workflows in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Archival, Aggregation, Collection, Performance

Description: Move aging documents from hot collections to archive collections in MongoDB using bulk operations and scheduled workflows to keep queries fast.

---

## Overview

As collections grow, older infrequently accessed documents slow down queries and increase index size. Archival workflows move these documents to a separate collection or database, keeping the hot collection lean while retaining historical data for compliance or reporting.

## Identifying Documents for Archival

Query the hot collection to find documents that meet archival criteria, such as being older than a threshold or having a terminal status.

```javascript
async function findArchivable(db, collection, thresholdDays = 180) {
  const cutoff = new Date(Date.now() - thresholdDays * 24 * 60 * 60 * 1000);
  return db.collection(collection).find(
    {
      createdAt: { $lt: cutoff },
      status: { $in: ["completed", "cancelled"] },
    },
    { projection: { _id: 1 } }
  ).toArray();
}
```

## Bulk Archive with Session

Use a client session to ensure the copy-then-delete operation is atomic.

```javascript
const { MongoClient } = require("mongodb");

async function archiveBatch(db, session, ids) {
  const oids = ids.map((id) => id._id);
  const source = db.collection("orders");
  const archive = db.collection("orders_archive");

  const docs = await source.find({ _id: { $in: oids } }).toArray();

  if (docs.length === 0) return 0;

  await session.withTransaction(async () => {
    const stamped = docs.map((d) => ({ ...d, archivedAt: new Date() }));
    await archive.insertMany(stamped, { session });
    await source.deleteMany({ _id: { $in: oids } }, { session });
  });

  return docs.length;
}
```

## Scheduled Archival Script

Run the archival workflow as a Node.js script triggered by a cron job or scheduler.

```javascript
async function runArchival() {
  const client = new MongoClient(process.env.MONGODB_URI);
  await client.connect();
  const db = client.db("myapp");
  const session = client.startSession();

  let total = 0;
  const batchSize = 500;

  try {
    let ids;
    do {
      ids = await findArchivable(db, "orders");
      const batch = ids.slice(0, batchSize);
      if (batch.length === 0) break;
      const moved = await archiveBatch(db, session, batch);
      total += moved;
      console.log(`Archived ${moved} documents (total: ${total})`);
    } while (ids.length >= batchSize);
  } finally {
    await session.endSession();
    await client.close();
  }

  console.log(`Archival complete. Total archived: ${total}`);
}

runArchival().catch(console.error);
```

## Cron Configuration

```bash
# Archive nightly at 2 AM
0 2 * * * /usr/bin/node /opt/scripts/archive-orders.js >> /var/log/archival.log 2>&1
```

## Partitioning Archive Collections by Period

For very large data volumes, partition archive collections by year or month to simplify retention management.

```javascript
function archiveCollectionName(date) {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  return `orders_archive_${y}_${m}`;
}

const archiveName = archiveCollectionName(new Date());
const archive = db.collection(archiveName);
```

## Verifying Archive Integrity

After archival, confirm counts match before considering the run complete.

```javascript
async function verifyArchival(db, archivedIds) {
  const archiveCount = await db.collection("orders_archive").countDocuments({
    _id: { $in: archivedIds },
  });
  const sourceCount = await db.collection("orders").countDocuments({
    _id: { $in: archivedIds },
  });
  if (archiveCount !== archivedIds.length || sourceCount !== 0) {
    throw new Error(
      `Archive mismatch: expected ${archivedIds.length}, archived ${archiveCount}, remaining ${sourceCount}`
    );
  }
}
```

## Summary

MongoDB document archival workflows move completed or old documents from active collections to archive collections using transactions to ensure consistency. Running archival in nightly batches with a session-based copy-then-delete pattern keeps the hot collection lean, improves query performance, and maintains a complete historical record for reporting and compliance.
