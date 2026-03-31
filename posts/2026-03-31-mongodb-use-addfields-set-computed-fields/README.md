# How to Use $addFields and $set to Add Computed Fields in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Computed Field

Description: Learn how to use $addFields and $set in MongoDB aggregation to add or overwrite fields with computed values while preserving all existing document fields.

---

## What Are $addFields and $set?

The `$addFields` stage adds new fields to documents or overwrites existing ones with computed values. All other fields in the document are preserved without being listed explicitly. `$set` is an alias for `$addFields` introduced in MongoDB 4.2 - they are identical in behavior.

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $addFields: {
      <newField>: <expression>,
      <existingField>: <expression>  // overwrites
    }
  }
])
```

## Example: Adding a Computed Field

```javascript
db.orders.aggregate([
  {
    $addFields: {
      totalWithTax: { $multiply: ["$subtotal", 1.08] }
    }
  }
])
// All original fields are kept; totalWithTax is added
```

## Example: Adding Multiple Fields

```javascript
db.products.aggregate([
  {
    $addFields: {
      discountAmount: { $multiply: ["$price", "$discountRate"] },
      finalPrice: {
        $subtract: [
          "$price",
          { $multiply: ["$price", "$discountRate"] }
        ]
      },
      inStockLabel: {
        $cond: { if: { $gt: ["$stock", 0] }, then: "In Stock", else: "Out of Stock" }
      }
    }
  }
])
```

## Overwriting an Existing Field

```javascript
db.users.aggregate([
  {
    $addFields: {
      // Convert email to lowercase
      email: { $toLower: "$email" }
    }
  }
])
```

## Adding a Field Based on Another Field's Type

```javascript
db.events.aggregate([
  {
    $addFields: {
      dateString: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } }
    }
  }
])
```

## Adding Nested Fields

```javascript
db.orders.aggregate([
  {
    $addFields: {
      "shipping.estimatedDays": {
        $cond: {
          if: { $eq: ["$shippingType", "express"] },
          then: 2,
          else: 7
        }
      }
    }
  }
])
```

## Using $addFields with $map on Arrays

```javascript
db.orders.aggregate([
  {
    $addFields: {
      items: {
        $map: {
          input: "$items",
          as: "item",
          in: {
            $mergeObjects: [
              "$$item",
              { lineTotal: { $multiply: ["$$item.price", "$$item.quantity"] } }
            ]
          }
        }
      }
    }
  }
])
```

This adds a `lineTotal` field to each item in the array.

## $addFields vs $project

| Behavior | $addFields / $set | $project |
|----------|-------------------|----------|
| Preserves all existing fields | Yes | No (must list explicitly) |
| Adds new computed fields | Yes | Yes |
| Overwrites fields | Yes | Yes |
| Use when | Augmenting documents | Reshaping output shape |

## Using $set (the Alias)

```javascript
db.inventory.aggregate([
  {
    $set: {
      lastUpdated: "$$NOW",
      category: { $toUpper: "$category" }
    }
  }
])
```

`$set` and `$addFields` are functionally identical. Many teams use `$set` in aggregation because it visually matches the `$set` update operator.

## Summary

The `$addFields` (or `$set`) stage is the preferred way to augment documents in a MongoDB aggregation pipeline without discarding existing fields. It supports the full expression language including arithmetic, string operations, conditionals, and array transformations. Use it whenever you need to compute derived fields while keeping the document intact.
