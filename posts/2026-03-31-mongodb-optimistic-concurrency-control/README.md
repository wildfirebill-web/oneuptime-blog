# How to Implement Optimistic Concurrency Control in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Concurrency, Optimistic Locking, Transaction, Conflict

Description: Learn how to implement optimistic concurrency control in MongoDB using version fields to detect and handle concurrent update conflicts without locking documents.

---

## Overview

Optimistic concurrency control (OCC) assumes conflicts are rare and allows concurrent reads without locking. When a write is committed, it checks whether the document was modified since the read. If it was, the write is rejected. This avoids the performance cost of pessimistic locking.

## The Version Field Pattern

Add a `version` field to documents that need OCC. Increment it on every update:

```javascript
// Initial document
{
  _id: ObjectId("..."),
  productId: "PROD-001",
  stock: 100,
  version: 1
}
```

## Implementing OCC with findOneAndUpdate

Read the document and capture the current version. When updating, include the version in the filter:

```javascript
async function decrementStock(db, productId, quantity) {
  // 1. Read the document and its current version
  const product = await db.collection("products").findOne({ productId });
  if (!product) throw new Error("Product not found");
  if (product.stock < quantity) throw new Error("Insufficient stock");

  // 2. Update only if the version has not changed since the read
  const result = await db.collection("products").findOneAndUpdate(
    {
      productId,
      version: product.version  // Optimistic lock check
    },
    {
      $inc: { stock: -quantity, version: 1 }
    },
    { returnDocument: "after" }
  );

  if (!result) {
    throw new Error("Conflict: document was modified concurrently, retry the operation");
  }

  return result;
}
```

If another process updated the document between the read and write, the filter `version: product.version` will not match and `findOneAndUpdate` returns `null`.

## Retry Logic for Conflicts

Wrap the operation in a retry loop to handle transient conflicts:

```javascript
async function decrementStockWithRetry(db, productId, quantity, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await decrementStock(db, productId, quantity);
    } catch (err) {
      if (err.message.includes("Conflict") && attempt < maxRetries) {
        const delay = attempt * 50 + Math.random() * 50;
        await new Promise(r => setTimeout(r, delay));
        console.log(`Retrying after conflict (attempt ${attempt})`);
      } else {
        throw err;
      }
    }
  }
}
```

## Using OCC with Mongoose

```javascript
const mongoose = require("mongoose");

const productSchema = new mongoose.Schema({
  productId: String,
  stock: Number,
  version: { type: Number, default: 0 }
});

const Product = mongoose.model("Product", productSchema);

async function decrementStockMongoose(productId, quantity) {
  const product = await Product.findOne({ productId });
  if (product.stock < quantity) throw new Error("Insufficient stock");

  const updated = await Product.findOneAndUpdate(
    { productId, version: product.version },
    { $inc: { stock: -quantity, version: 1 } },
    { new: true }
  );

  if (!updated) throw new Error("Conflict detected, retry needed");
  return updated;
}
```

## When to Use Optimistic vs Pessimistic Concurrency

Use OCC when conflicts are rare (most reads result in successful writes). Use MongoDB transactions (pessimistic) when conflicts are frequent or when you need to update multiple documents atomically and cannot tolerate retries.

## Summary

Optimistic concurrency control in MongoDB uses a `version` field that is included in update filters to detect concurrent modifications. If the version does not match, the update returns null, signaling a conflict. Wrap operations in retry loops with jitter to handle transient conflicts gracefully. OCC is best suited for low-conflict scenarios like inventory systems, user profile updates, and configuration management.
