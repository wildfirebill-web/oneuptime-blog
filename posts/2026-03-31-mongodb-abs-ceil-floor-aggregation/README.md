# How to Use $abs and $ceil and $floor in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Math, $abs, $ceil, $floor, Pipeline

Description: Learn how to use MongoDB's $abs, $ceil, and $floor arithmetic aggregation operators to round, normalize, and transform numeric fields in aggregation pipelines.

---

## Overview

MongoDB's aggregation framework provides several math operators for numeric transformations. `$abs`, `$ceil`, and `$floor` are essential for working with decimals and signed numbers - rounding values, taking absolute differences, and normalizing floating-point results in computed fields.

## Sample Data

For this guide, use a `transactions` collection:

```javascript
db.transactions.insertMany([
  { _id: 1, item: "keyboard", amount: 49.95, discount: -5.50, quantity: 3 },
  { _id: 2, item: "monitor", amount: 299.10, discount: -25.00, quantity: 1 },
  { _id: 3, item: "mouse", amount: 19.75, discount: -2.25, quantity: 5 },
  { _id: 4, item: "headset", amount: -15.00, discount: 0, quantity: 2 }
]);
```

## Using $abs

`$abs` returns the absolute value of a number, removing any negative sign.

### Syntax

```javascript
{ $abs: <expression> }
```

### Example: Get Absolute Discount Amount

```javascript
db.transactions.aggregate([
  {
    $project: {
      item: 1,
      amount: 1,
      discountAmount: { $abs: "$discount" }
    }
  }
]);
```

```text
{ item: "keyboard", amount: 49.95, discountAmount: 5.5 }
{ item: "monitor", amount: 299.10, discountAmount: 25 }
{ item: "mouse", amount: 19.75, discountAmount: 2.25 }
{ item: "headset", amount: -15, discountAmount: 0 }
```

### Example: Absolute Difference Between Two Fields

```javascript
db.transactions.aggregate([
  {
    $project: {
      item: 1,
      netAmount: { $abs: { $subtract: ["$amount", "$discount"] } }
    }
  }
]);
```

`$abs` is useful when computing distances, deviations, or differences where the sign does not matter.

## Using $ceil

`$ceil` rounds a number **up** to the nearest integer (ceiling function).

### Syntax

```javascript
{ $ceil: <expression> }
```

### Example: Round Up Prices for Display

```javascript
db.transactions.aggregate([
  {
    $project: {
      item: 1,
      amount: 1,
      roundedUp: { $ceil: "$amount" }
    }
  }
]);
```

```text
{ item: "keyboard", amount: 49.95, roundedUp: 50 }
{ item: "monitor", amount: 299.10, roundedUp: 300 }
{ item: "mouse", amount: 19.75, roundedUp: 20 }
{ item: "headset", amount: -15, roundedUp: -15 }
```

Note: `$ceil` on a negative number rounds toward zero (-15.3 becomes -15, not -16).

### Example: Calculate Pages for Pagination

```javascript
db.reports.aggregate([
  {
    $project: {
      totalRecords: 1,
      pageSize: { $literal: 10 },
      totalPages: {
        $ceil: {
          $divide: ["$totalRecords", 10]
        }
      }
    }
  }
]);
```

## Using $floor

`$floor` rounds a number **down** to the nearest integer (floor function).

### Syntax

```javascript
{ $floor: <expression> }
```

### Example: Floor Prices for Discounted Labels

```javascript
db.transactions.aggregate([
  {
    $project: {
      item: 1,
      amount: 1,
      roundedDown: { $floor: "$amount" }
    }
  }
]);
```

```text
{ item: "keyboard", amount: 49.95, roundedDown: 49 }
{ item: "monitor", amount: 299.10, roundedDown: 299 }
{ item: "mouse", amount: 19.75, roundedDown: 19 }
{ item: "headset", amount: -15, roundedDown: -15 }
```

## Combining $abs, $ceil, and $floor

### Example: Revenue Analysis with Rounded Metrics

```javascript
db.transactions.aggregate([
  {
    $project: {
      item: 1,
      grossAmount: { $abs: "$amount" },
      discountApplied: { $abs: "$discount" },
      netRevenue: {
        $abs: { $subtract: ["$amount", "$discount"] }
      }
    }
  },
  {
    $project: {
      item: 1,
      grossAmount: 1,
      discountApplied: 1,
      netRevenue: 1,
      netRevenueFloor: { $floor: "$netRevenue" },
      netRevenueCeil: { $ceil: "$netRevenue" }
    }
  },
  {
    $group: {
      _id: null,
      totalGross: { $sum: "$grossAmount" },
      totalDiscount: { $sum: "$discountApplied" },
      totalNetFloor: { $sum: "$netRevenueFloor" },
      totalNetCeil: { $sum: "$netRevenueCeil" }
    }
  }
]);
```

### Example: Bucket Items by Floored Price Range

```javascript
db.transactions.aggregate([
  {
    $addFields: {
      priceRange: {
        $multiply: [
          { $floor: { $divide: ["$amount", 50] } },
          50
        ]
      }
    }
  },
  {
    $group: {
      _id: "$priceRange",
      count: { $sum: 1 },
      items: { $push: "$item" }
    }
  },
  { $sort: { _id: 1 } }
]);
```

```text
{ _id: 0, count: 2, items: ["mouse", "headset"] }
{ _id: 0, count: 1, items: ["keyboard"] }
{ _id: 250, count: 1, items: ["monitor"] }
```

This groups items into $0-$49, $50-$99, ..., $250-$299 price buckets.

## Handling Null and Non-Numeric Values

All three operators return `null` if the input expression evaluates to `null` or a missing field. They throw an error for non-numeric types.

```javascript
db.transactions.aggregate([
  {
    $project: {
      safeAbs: {
        $abs: {
          $ifNull: ["$amount", 0]
        }
      },
      safeCeil: {
        $ceil: {
          $ifNull: ["$amount", 0]
        }
      }
    }
  }
]);
```

## Summary

MongoDB's `$abs`, `$ceil`, and `$floor` operators enable precise numeric transformations in aggregation pipelines. Use `$abs` to normalize signed values like discounts or differences, `$ceil` to round up for conservative estimates or pagination calculations, and `$floor` to round down for price buckets or conservative revenue reporting. These operators compose naturally with other arithmetic expressions like `$divide`, `$multiply`, and `$subtract`.
