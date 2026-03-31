# How to Use $inc to Increment or Decrement Numeric Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $inc, Update Operator, Counter, Numeric

Description: Learn how to use MongoDB's $inc operator to atomically increment or decrement numeric fields, with practical examples for counters and inventory.

---

## What Is $inc

The `$inc` operator increments or decrements a numeric field by a specified amount in a single atomic operation. If the field does not exist, `$inc` creates it and sets it to the specified value.

Syntax:

```javascript
{ $inc: { field: amount } }
```

Use a positive `amount` to increment, negative to decrement.

## Basic Examples

```javascript
// Increment page view count by 1
db.articles.updateOne(
  { _id: articleId },
  { $inc: { viewCount: 1 } }
)

// Decrement stock by the ordered quantity
db.products.updateOne(
  { _id: productId },
  { $inc: { stock: -quantity } }
)
```

## Why Use $inc Instead of $set

`$inc` is atomic - it reads and updates the field in one server-side operation. Using `$set` with a client-computed value is not atomic:

```javascript
// UNSAFE - race condition between read and write
const product = await collection.findOne({ _id: productId });
await collection.updateOne(
  { _id: productId },
  { $set: { stock: product.stock - 1 } }
);

// SAFE - atomic server-side increment
await collection.updateOne(
  { _id: productId },
  { $inc: { stock: -1 } }
);
```

## Incrementing Multiple Fields at Once

```javascript
// Track both total views and unique views
db.pageStats.updateOne(
  { page: "/home" },
  { $inc: { totalViews: 1, uniqueViews: isNewVisitor ? 1 : 0 } }
)
```

Or always increment both:

```javascript
db.userStats.updateOne(
  { userId: "u42" },
  { $inc: { loginCount: 1, sessionCount: 1 } }
)
```

## Using $inc with Upsert for Counters

Combine `$inc` with `upsert: true` to create or update a counter document:

```javascript
await db.collection("dailyStats").updateOne(
  { date: "2025-03-31", event: "page_view" },
  { $inc: { count: 1 } },
  { upsert: true }
)
```

The first time this runs for a given date/event, it creates the document with `count: 1`. Subsequent calls increment `count`.

## Combining $inc with $set

Combine operators in a single update:

```javascript
db.orders.updateOne(
  { _id: orderId },
  {
    $inc: { itemCount: 1, totalCost: itemPrice },
    $set: { updatedAt: new Date() }
  }
)
```

## $inc with Nested Fields

Use dot notation:

```javascript
db.products.updateOne(
  { _id: productId },
  { $inc: { "stats.sold": 1, "stats.revenue": salePrice } }
)
```

## Inventory Management Example

```javascript
async function processOrder(db, productId, quantity) {
  const result = await db.collection("products").updateOne(
    {
      _id: productId,
      stock: { $gte: quantity }  // only update if enough stock
    },
    { $inc: { stock: -quantity, reservedCount: quantity } }
  );

  if (result.modifiedCount === 0) {
    throw new Error("Insufficient stock");
  }
  return result;
}
```

## Common Mistakes

- Using `$inc` with a non-numeric value - MongoDB will throw an error.
- Using `$set` for counters and creating race conditions in concurrent environments.
- Forgetting that `$inc` creates the field if missing, initializing it to the increment value.

## Summary

`$inc` atomically increments or decrements numeric fields by a specified amount, making it safe for counters and inventory management in concurrent environments. Use positive values to increment and negative values to decrement. Combine with `upsert: true` for automatic counter creation, and pair with `$set` in the same update for additional field changes.
