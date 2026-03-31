# How to Implement Idempotent Operations in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Idempotent, Reliability, Deduplication, Transaction

Description: Learn how to implement idempotent operations in MongoDB using unique identifiers, upserts, and deduplication collections to safely retry failed operations.

---

## Overview

An idempotent operation produces the same result whether it is executed once or multiple times. This is critical for distributed systems where network failures can cause retries that accidentally duplicate inserts or apply updates twice. MongoDB provides several patterns for achieving idempotency.

## Pattern 1: Idempotent Inserts with a Unique ID

The simplest approach is to provide a deterministic document ID. If the insert is retried, MongoDB's duplicate key error indicates the document already exists:

```javascript
const { v5: uuidv5 } = require("uuid");

const NAMESPACE = "6ba7b810-9dad-11d1-80b4-00c04fd430c8";

async function createOrderIdempotent(db, orderData) {
  const idempotencyKey = uuidv5(
    `${orderData.userId}-${orderData.externalOrderId}`,
    NAMESPACE
  );

  try {
    await db.collection("orders").insertOne({
      _id: idempotencyKey,
      ...orderData,
      createdAt: new Date()
    });
    return { created: true };
  } catch (err) {
    if (err.code === 11000) {
      // Document already exists - safe to return success
      return { created: false, duplicate: true };
    }
    throw err;
  }
}
```

## Pattern 2: Idempotent Upserts

Use `updateOne` with `upsert: true` for operations that should be applied exactly once based on a business key:

```javascript
async function processPaymentIdempotent(db, paymentId, amount, userId) {
  const result = await db.collection("payments").updateOne(
    { paymentId },  // Unique business key
    {
      $setOnInsert: {
        paymentId,
        amount,
        userId,
        status: "processed",
        processedAt: new Date()
      }
    },
    { upsert: true }
  );

  if (result.upsertedCount === 1) {
    return { status: "created" };
  } else {
    return { status: "already_processed" };
  }
}
```

`$setOnInsert` only applies the fields when a new document is inserted, leaving existing documents unchanged.

## Pattern 3: Deduplication Collection

Store idempotency keys in a separate collection with a TTL index for automatic cleanup:

```javascript
// Create dedup collection with TTL
db.idempotency_keys.createIndex({ key: 1 }, { unique: true })
db.idempotency_keys.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })

async function executeIdempotent(db, idempotencyKey, operation) {
  try {
    await db.collection("idempotency_keys").insertOne({
      key: idempotencyKey,
      createdAt: new Date()
    });
  } catch (err) {
    if (err.code === 11000) {
      console.log(`Operation ${idempotencyKey} already executed`);
      return { duplicate: true };
    }
    throw err;
  }

  // Key did not exist, safe to execute the operation
  return { result: await operation(), duplicate: false };
}

// Usage
await executeIdempotent(db, `send-email-${userId}-${campaignId}`, async () => {
  await sendEmail(userId, campaignId);
  await db.collection("email_log").insertOne({ userId, campaignId, sentAt: new Date() });
});
```

## Pattern 4: Conditional Updates with $set

For updates that should only apply if the state matches expectations:

```javascript
async function shipOrderIdempotent(db, orderId, trackingNumber) {
  const result = await db.collection("orders").updateOne(
    { _id: orderId, status: "paid" },  // Only update if still in 'paid' state
    {
      $set: {
        status: "shipped",
        trackingNumber,
        shippedAt: new Date()
      }
    }
  );

  if (result.matchedCount === 0) {
    const order = await db.collection("orders").findOne({ _id: orderId });
    if (order?.status === "shipped") return { status: "already_shipped" };
    throw new Error(`Order ${orderId} cannot be shipped from status: ${order?.status}`);
  }

  return { status: "shipped" };
}
```

## Summary

Implement idempotent MongoDB operations using deterministic IDs with duplicate key error handling, `$setOnInsert` upserts for business-key-based deduplication, or a separate idempotency_keys collection with a TTL. Always design retryable operations around one of these patterns to make your application resilient to network failures and duplicate message delivery.
