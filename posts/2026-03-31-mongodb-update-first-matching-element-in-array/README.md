# How to Update the First Matching Element in an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Array, Positional Operator, Operator

Description: Learn how to update the first element in a MongoDB array that matches a condition using the $ positional operator in updateOne and updateMany.

---

The positional `$` operator is MongoDB's solution for updating the first array element that matches a query condition. It captures the index of the first matching array element from the filter and uses that position in the update path.

## How the Positional Operator Works

The `$` placeholder in an update path refers to the matched array element's index as determined by the query filter:

```javascript
db.orders.updateOne(
  { _id: orderId, "items.sku": "B200" },  // query identifies the element
  { $set: { "items.$.status": "shipped" } } // $ refers to matched element
)
```

The filter must contain an array condition that identifies the element. The `$` in the update path is replaced by the matched index.

## Sample Data

```javascript
db.carts.insertOne({
  _id: 1,
  userId: "user-42",
  items: [
    { sku: "A100", qty: 2, reserved: false },
    { sku: "B200", qty: 1, reserved: false },
    { sku: "C300", qty: 4, reserved: true }
  ]
})
```

## Basic Positional Update

```javascript
// Mark the B200 item as reserved
db.carts.updateOne(
  { _id: 1, "items.sku": "B200" },
  { $set: { "items.$.reserved": true } }
)
```

Only the first element with `sku: "B200"` is updated.

## Updating Multiple Fields of the Matched Element

```javascript
db.carts.updateOne(
  { _id: 1, "items.sku": "A100" },
  {
    $set: {
      "items.$.qty": 5,
      "items.$.reserved": true,
      "items.$.updatedAt": new Date()
    }
  }
)
```

## Using $inc on a Matched Element

```javascript
// Increment quantity of a specific cart item
db.carts.updateOne(
  { userId: "user-42", "items.sku": "A100" },
  { $inc: { "items.$.qty": 1 } }
)
```

## Positional Operator with Nested Conditions

When the array condition uses multiple fields via `$elemMatch`, the positional operator still works:

```javascript
db.carts.updateOne(
  {
    _id: 1,
    items: { $elemMatch: { sku: "B200", reserved: false } }
  },
  { $set: { "items.$.reserved": true } }
)
```

## Cross-Document Updates (updateMany)

The positional `$` operator works with `updateMany` to update the first matching element in each document:

```javascript
// In all carts, mark the first item with sku "PROMO" as discounted
db.carts.updateMany(
  { "items.sku": "PROMO" },
  { $set: { "items.$.discounted": true } }
)
```

## Limitations of the Positional Operator

The `$` operator has two important limitations:
1. It updates only the first matching element - use `$[identifier]` with `arrayFilters` to update all matching elements
2. It cannot be used to update elements in nested arrays (arrays within arrays)

```javascript
// To update ALL matching elements, use $[identifier] instead
db.carts.updateMany(
  { "items.reserved": false },
  { $set: { "items.$[item].reserved": true } },
  { arrayFilters: [{ "item.reserved": false }] }
)
```

## Verify the Update

```javascript
const result = await db.collection("carts").updateOne(
  { userId: "user-42", "items.sku": "B200" },
  { $set: { "items.$.qty": 3 } }
);

console.log(`Matched: ${result.matchedCount}, Modified: ${result.modifiedCount}`);
```

## Summary

The `$` positional operator updates the first array element matching the query filter condition. Include the array field condition in the filter, then use `$` as the index placeholder in the update path. It works with all field update operators (`$set`, `$inc`, `$unset`, etc.) and with `updateMany` across multiple documents. When you need to update all matching elements rather than just the first, use the `$[identifier]` filtered positional operator with `arrayFilters`.
