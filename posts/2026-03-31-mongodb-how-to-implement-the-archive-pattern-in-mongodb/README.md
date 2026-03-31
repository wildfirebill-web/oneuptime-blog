# How to Implement the Archive Pattern in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Modeling, Archive Pattern, Data Lifecycle, Performance

Description: Learn how to implement the archive pattern in MongoDB to move stale or infrequently accessed documents to a separate collection, keeping your hot collection small and fast.

---

## What Is the Archive Pattern?

The archive pattern moves old or infrequently accessed documents from the active ("hot") collection to an archive ("cold") collection. This keeps the active collection small, which improves query performance, reduces index size, and lowers memory usage for the working dataset.

## When to Use the Archive Pattern

Use archiving when:
- Documents have a natural lifecycle (active -> inactive -> archived)
- Old data is rarely queried but must be retained for compliance
- The active collection grows unboundedly and query performance degrades
- You want to apply different retention policies to different data ages

## Data Structure

Active and archive collections share the same document structure, with optional extra metadata in the archive:

```javascript
// Active collection: orders
{
  _id: ObjectId("ord001"),
  customerId: ObjectId("cust123"),
  status: "completed",
  total: 199.99,
  createdAt: ISODate("2024-01-15T00:00:00Z"),
  completedAt: ISODate("2024-01-20T00:00:00Z")
}

// Archive collection: ordersArchive
{
  _id: ObjectId("ord001"),   // Same _id preserved
  customerId: ObjectId("cust123"),
  status: "completed",
  total: 199.99,
  createdAt: ISODate("2024-01-15T00:00:00Z"),
  completedAt: ISODate("2024-01-20T00:00:00Z"),
  // Archive-specific metadata
  archivedAt: ISODate("2026-01-01T00:00:00Z"),
  archiveReason: "age_policy"
}
```

## Archiving Script

Move documents older than 2 years from orders to ordersArchive:

```javascript
async function archiveOldOrders() {
  const cutoffDate = new Date();
  cutoffDate.setFullYear(cutoffDate.getFullYear() - 2);

  const BATCH_SIZE = 1000;
  let archivedCount = 0;

  while (true) {
    // Find a batch of old completed orders
    const batch = await db.orders.find({
      status: "completed",
      completedAt: { $lt: cutoffDate }
    }).limit(BATCH_SIZE).toArray();

    if (batch.length === 0) break;

    // Add archive metadata
    const archiveDocs = batch.map(doc => ({
      ...doc,
      archivedAt: new Date(),
      archiveReason: "age_policy"
    }));

    // Insert into archive collection
    await db.ordersArchive.insertMany(archiveDocs, { ordered: false });

    // Delete from active collection
    const ids = batch.map(doc => doc._id);
    await db.orders.deleteMany({ _id: { $in: ids } });

    archivedCount += batch.length;
    print(`Archived ${archivedCount} orders so far...`);
  }

  print(`Archiving complete. Total archived: ${archivedCount}`);
}
```

## Using Transactions for Safe Archiving

For critical data, use transactions to ensure no data is lost during the move:

```javascript
async function archiveOrderSafely(orderId) {
  const session = db.getMongo().startSession();
  session.startTransaction();

  try {
    const order = await db.orders.findOne({ _id: orderId }, { session });
    if (!order) throw new Error("Order not found");

    await db.ordersArchive.insertOne(
      { ...order, archivedAt: new Date() },
      { session }
    );

    await db.orders.deleteOne({ _id: orderId }, { session });

    await session.commitTransaction();
  } catch (e) {
    await session.abortTransaction();
    throw e;
  } finally {
    session.endSession();
  }
}
```

## Application-Level Transparent Archiving

Build a query layer that searches both collections:

```javascript
async function findOrder(orderId) {
  // Check active collection first
  let order = await db.orders.findOne({ _id: orderId });

  // Fall back to archive if not found
  if (!order) {
    order = await db.ordersArchive.findOne({ _id: orderId });
    if (order) order._isArchived = true;
  }

  return order;
}

async function searchOrders(customerId, includeArchived = false) {
  const query = { customerId: ObjectId(customerId) };
  const activeOrders = await db.orders.find(query).toArray();

  if (!includeArchived) return activeOrders;

  const archivedOrders = await db.ordersArchive.find(query).toArray();
  return [...activeOrders, ...archivedOrders];
}
```

## Scheduling Archival with a TTL Index (Alternative)

For simple time-based archival where you can delete rather than move:

```javascript
// Auto-delete completed orders after 90 days (no separate archive)
db.orders.createIndex(
  { completedAt: 1 },
  { expireAfterSeconds: 7776000,  // 90 days
    partialFilterExpression: { status: "completed" } }
)
```

## Indexing the Archive Collection

The archive collection needs its own indexes for compliance and audit queries:

```javascript
db.ordersArchive.createIndex({ customerId: 1, archivedAt: -1 })
db.ordersArchive.createIndex({ completedAt: 1 })
db.ordersArchive.createIndex({ archivedAt: 1 })
```

## Monitoring Active Collection Size

```javascript
// Check how many orders are ready to archive
db.orders.countDocuments({
  status: "completed",
  completedAt: { $lt: new Date(Date.now() - 2 * 365 * 24 * 3600 * 1000) }
})

// Track active vs archived ratio
print("Active:", db.orders.estimatedDocumentCount());
print("Archived:", db.ordersArchive.estimatedDocumentCount());
```

## Summary

The archive pattern moves stale documents from the active collection to an archive collection to keep query performance high and working set small. Implement archiving in batches with periodic background jobs, use transactions for critical data integrity, and build a transparent access layer that queries both collections when historical lookups are needed. Index the archive collection independently for compliance and audit queries, and schedule archiving jobs during off-peak hours.
