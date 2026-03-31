# How to Implement Audit Logging in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Audit, Logging, Security, Compliance

Description: Learn how to implement audit logging in MongoDB at the server level and application level to track who accessed and modified data for security and compliance requirements.

---

## Overview

Audit logging tracks database operations for security investigations, compliance (SOC 2, HIPAA, GDPR), and debugging. MongoDB Enterprise includes native audit logging. For Community Edition, you can implement application-level audit logging using change streams or middleware.

## MongoDB Enterprise Native Audit Logging

Enable audit logging in `mongod.conf`:

```yaml
security:
  authorization: enabled

auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{
    atype: {
      $in: [
        "authenticate",
        "authCheck",
        "createCollection",
        "dropCollection",
        "createDatabase",
        "dropDatabase",
        "find",
        "insert",
        "remove",
        "update"
      ]
    }
  }'
```

## Application-Level Audit Log Schema

```javascript
// audit_logs collection
{
  _id: ObjectId(),
  timestamp: new Date(),
  userId: "user-123",
  userEmail: "alice@example.com",
  action: "UPDATE",
  collection: "orders",
  documentId: "ORD-456",
  before: { status: "pending", total: 149.99 },
  after: { status: "shipped", total: 149.99 },
  ip: "192.168.1.100",
  userAgent: "Mozilla/5.0...",
  requestId: "req-789"
}
```

## Middleware-Based Audit Logging

```javascript
function auditMiddleware(action, collectionName) {
  return async (req, res, next) => {
    const originalJson = res.json.bind(res);

    res.json = function(data) {
      logAuditEvent(req, action, collectionName, data).catch(console.error);
      return originalJson(data);
    };

    next();
  };
}

async function logAuditEvent(req, action, collectionName, responseData) {
  await db.collection("audit_logs").insertOne({
    timestamp: new Date(),
    userId: req.user?.id,
    userEmail: req.user?.email,
    action,
    collection: collectionName,
    documentId: req.params.id || responseData?._id,
    requestBody: action !== "READ" ? req.body : undefined,
    ip: req.ip,
    userAgent: req.get("user-agent"),
    requestId: req.id
  });
}

// Apply to routes
app.put("/api/orders/:id",
  authMiddleware,
  auditMiddleware("UPDATE", "orders"),
  updateOrderHandler
);
```

## Capturing Before/After Values

For full change tracking, read the document before the update:

```javascript
async function updateWithAudit(db, collection, filter, update, userId) {
  const before = await db.collection(collection).findOne(filter);
  const result = await db.collection(collection).findOneAndUpdate(
    filter,
    update,
    { returnDocument: "after" }
  );

  await db.collection("audit_logs").insertOne({
    timestamp: new Date(),
    userId,
    action: "UPDATE",
    collection,
    documentId: before._id,
    before,
    after: result,
  });

  return result;
}
```

## Setting Up a TTL Index on Audit Logs

Automatically expire old audit log entries:

```javascript
// Expire audit logs after 365 days
db.audit_logs.createIndex({ timestamp: 1 }, { expireAfterSeconds: 31536000 })
```

## Summary

Implement audit logging in MongoDB using native Enterprise audit logging for production security requirements or application-level middleware for Community Edition. Store structured audit documents with before/after values, user identity, and request metadata. Add a TTL index to automatically purge logs older than your retention policy requires.
