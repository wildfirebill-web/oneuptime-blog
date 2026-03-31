# How to Use $mul to Multiply Field Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $mul, Update Operator, Arithmetic, Numeric

Description: Learn how to use MongoDB's $mul operator to multiply a numeric field by a factor atomically, with examples for price adjustments and bulk scaling.

---

## What Is $mul

The `$mul` operator multiplies the value of a field by a specified number in a single atomic server-side operation. It is the multiplication equivalent of `$inc`.

Syntax:

```javascript
{ $mul: { field: multiplier } }
```

If the field does not exist, `$mul` creates it with a value of `0` (the product of 0 times any number is 0, then multiplied by the given factor - actually MongoDB sets it to 0 when the field is missing).

## Basic Examples

```javascript
// Apply a 10% discount (multiply price by 0.9)
db.products.updateOne(
  { _id: productId },
  { $mul: { price: 0.9 } }
)

// Double the quantity
db.inventory.updateOne(
  { _id: itemId },
  { $mul: { qty: 2 } }
)
```

## Bulk Price Adjustment

Apply a price increase to a category of products:

```javascript
// Raise all electronics prices by 5%
await db.collection("products").updateMany(
  { category: "electronics" },
  { $mul: { price: 1.05 } }
)
```

## Applying a Discount to All Items in a Subcategory

```javascript
// 20% off all sale items
await db.collection("products").updateMany(
  { tags: "sale" },
  { $mul: { price: 0.80 } }
)
```

## Combining $mul with Other Update Operators

```javascript
db.products.updateOne(
  { _id: productId },
  {
    $mul: { price: 1.1 },             // increase price by 10%
    $set: { lastPriceUpdate: new Date() },
    $inc: { priceChangeCount: 1 }
  }
)
```

## $mul vs Manual Computation with $set

Like `$inc`, `$mul` avoids the read-compute-write race condition:

```javascript
// UNSAFE - race condition
const doc = await collection.findOne({ _id: id });
await collection.updateOne({ _id: id }, { $set: { price: doc.price * 1.1 } });

// SAFE - atomic
await collection.updateOne({ _id: id }, { $mul: { price: 1.1 } });
```

## Handling Multiplier Edge Cases

```javascript
// Multiply by 0 - sets field to 0
db.products.updateOne({ _id: id }, { $mul: { price: 0 } })

// Multiply by 1 - no change (but does touch the document)
db.products.updateOne({ _id: id }, { $mul: { price: 1 } })

// Multiply by -1 - negates the value
db.products.updateOne({ _id: id }, { $mul: { discount: -1 } })
```

## Missing Fields Behavior

If the field does not exist, `$mul` initializes it to `0` (not the multiplier value):

```javascript
// Field "score" doesn't exist
db.players.updateOne({ _id: id }, { $mul: { score: 5 } })
// score is now 0 (not 5)

// Compare with $inc - missing field initializes to the increment value
db.players.updateOne({ _id: id }, { $inc: { score: 5 } })
// score is now 5
```

## Currency Rounding After $mul

Floating-point multiplication can introduce precision errors. Round the result using an aggregation pipeline update (MongoDB 4.2+):

```javascript
await db.collection("products").updateMany(
  { category: "apparel" },
  [
    {
      $set: {
        price: {
          $round: [{ $multiply: ["$price", 1.08] }, 2]
        }
      }
    }
  ]
)
```

## Common Mistakes

- Forgetting that `$mul` on a missing field initializes to `0`, not the multiplier.
- Using `$mul` on non-numeric fields - MongoDB returns an error.
- Applying repeated `$mul` multiplications without rounding, accumulating floating-point drift.

## Summary

`$mul` atomically multiplies a numeric field by a specified factor, making it safe for concurrent price adjustments and scaling operations. Unlike a `$set` with client-computed values, `$mul` is a server-side atomic operation. Remember that applying `$mul` to a non-existent field initializes it to `0`. For currency fields, follow `$mul` with rounding using an aggregation pipeline update.
