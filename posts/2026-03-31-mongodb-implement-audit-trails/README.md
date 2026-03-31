# How to Implement Audit Trails in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Audit, Trail, History, Compliance

Description: Learn how to implement audit trails in MongoDB to maintain a complete change history for documents, supporting compliance, debugging, and rollback requirements.

---

## Overview

An audit trail is a chronological record of every change made to a document, including who made the change, when, and what it was. Unlike audit logging (which captures events), an audit trail preserves full document history. MongoDB supports this through several patterns.

## Pattern 1: Versioned Documents Collection

Store a snapshot of the document every time it changes:

```javascript
// current state in main collection
{
  _id: "order-123",
  status: "shipped",
  total: 149.99,
  updatedAt: new Date(),
  updatedBy: "user-42"
}

// audit_trail collection - historical snapshots
{
  _id: ObjectId(),
  documentId: "order-123",
  collection: "orders",
  version: 3,
  snapshot: { status: "paid", total: 149.99 },
  changedFields: { status: "shipped" },
  changedBy: "user-42",
  changedAt: new Date()
}
```

## Pattern 2: Automatic Audit Trail with Mongoose

```javascript
const mongoose = require("mongoose");

const orderSchema = new mongoose.Schema({
  status: String,
  total: Number,
  updatedBy: String,
  updatedAt: Date
});

// Pre-save hook to record audit trail
orderSchema.pre("save", async function(next) {
  if (!this.isNew && this.isModified()) {
    const modifiedFields = {};
    this.modifiedPaths().forEach(path => {
      modifiedFields[path] = this.get(path);
    });

    await mongoose.model("AuditTrail").create({
      documentId: this._id,
      collection: "orders",
      changedFields: modifiedFields,
      snapshot: this.toObject(),
      changedBy: this.updatedBy,
      changedAt: new Date()
    });
  }
  next();
});
```

## Pattern 3: Embedded History Array

Embed a limited history directly in the document:

```javascript
async function updateOrderWithHistory(db, orderId, changes, userId) {
  const result = await db.collection("orders").findOneAndUpdate(
    { _id: orderId },
    {
      $set: changes,
      $push: {
        history: {
          $each: [{
            changes,
            changedBy: userId,
            changedAt: new Date()
          }],
          $slice: -50  // Keep only last 50 history entries
        }
      }
    },
    { returnDocument: "after" }
  );
  return result;
}
```

## Querying the Audit Trail

```javascript
// Get full history for a document
async function getDocumentHistory(db, collection, documentId) {
  return db.collection("audit_trails")
    .find({ documentId, collection })
    .sort({ changedAt: -1 })
    .toArray();
}

// Get all changes made by a specific user
async function getChangedByUser(db, userId, since) {
  return db.collection("audit_trails")
    .find({
      changedBy: userId,
      changedAt: { $gte: since }
    })
    .sort({ changedAt: -1 })
    .toArray();
}

// Find when a field was last changed
async function findLastFieldChange(db, documentId, fieldName) {
  return db.collection("audit_trails").findOne(
    { documentId, [`changedFields.${fieldName}`]: { $exists: true } },
    { sort: { changedAt: -1 } }
  );
}
```

## Setting Up Indexes

```javascript
db.audit_trails.createIndex({ documentId: 1, changedAt: -1 })
db.audit_trails.createIndex({ collection: 1, changedAt: -1 })
db.audit_trails.createIndex({ changedBy: 1, changedAt: -1 })

// TTL if you only need to retain history for a fixed period
db.audit_trails.createIndex(
  { changedAt: 1 },
  { expireAfterSeconds: 2 * 365 * 24 * 60 * 60 }  // 2 years
)
```

## Rollback Using Audit Trail

```javascript
async function rollbackToVersion(db, collection, documentId, version) {
  const auditEntry = await db.collection("audit_trails").findOne(
    { documentId, collection, version }
  );
  if (!auditEntry) throw new Error("Version not found");

  await db.collection(collection).replaceOne(
    { _id: documentId },
    auditEntry.snapshot
  );

  // Record the rollback itself
  await db.collection("audit_trails").insertOne({
    documentId,
    collection,
    version: auditEntry.version + 1,
    action: "rollback",
    rolledBackTo: version,
    snapshot: auditEntry.snapshot,
    changedAt: new Date()
  });
}
```

## Summary

Implement audit trails in MongoDB by storing document snapshots in a separate `audit_trails` collection using pre-save hooks or application-level wrappers. Index by `documentId` and `changedAt` for efficient history queries. Store full snapshots alongside changed fields to enable both diff views and point-in-time rollbacks. Add a TTL index if your compliance policy requires automatic expiry after a retention period.
