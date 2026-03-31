# How to Use $abs, $ceil, and $floor in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $abs, $ceil, $floor, Math Expressions, NoSQL

Description: Learn how to use MongoDB's $abs, $ceil, and $floor aggregation expression operators to compute absolute values, ceiling, and floor of numeric fields.

---

## Overview

MongoDB provides several math expression operators for numeric transformations in aggregation pipelines:

- `$abs` - returns the absolute value (removes negative sign)
- `$ceil` - rounds a number up to the nearest integer
- `$floor` - rounds a number down to the nearest integer
- `$round` - rounds to a specified number of decimal places
- `$trunc` - truncates decimal digits without rounding

## $abs - Absolute Value

Returns the absolute (non-negative) value of a number.

```javascript
{ $abs: expression }
```

### Example

```javascript
db.transactions.aggregate([
  {
    $project: {
      amount: 1,
      absoluteAmount: { $abs: "$amount" }
    }
  }
])
```

Input/output:

```javascript
{ amount: -150 }  ->  { amount: -150, absoluteAmount: 150 }
{ amount: 200  }  ->  { amount: 200,  absoluteAmount: 200 }
{ amount: 0    }  ->  { amount: 0,    absoluteAmount: 0   }
```

### Use Case - Distance Calculation

```javascript
db.priceChanges.aggregate([
  {
    $addFields: {
      priceChange: { $subtract: ["$newPrice", "$oldPrice"] },
      priceChangeAbs: { $abs: { $subtract: ["$newPrice", "$oldPrice"] } }
    }
  },
  { $sort: { priceChangeAbs: -1 } }
])
```

Sorts products by the magnitude of price change, regardless of direction.

## $ceil - Ceiling

Rounds a number up to the nearest integer.

```javascript
{ $ceil: expression }
```

### Example

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      weight: "$weightKg",
      shippingUnits: { $ceil: "$weightKg" }  // always round up for shipping
    }
  }
])
```

```javascript
{ weightKg: 2.1 }  ->  { shippingUnits: 3 }
{ weightKg: 2.0 }  ->  { shippingUnits: 2 }
{ weightKg: 1.9 }  ->  { shippingUnits: 2 }
```

### Use Case - Page Count

```javascript
db.datasets.aggregate([
  {
    $addFields: {
      totalPages: {
        $ceil: { $divide: ["$recordCount", 100] }
      }
    }
  }
])
```

Computes the number of pages needed for pagination.

## $floor - Floor

Rounds a number down to the nearest integer.

```javascript
{ $floor: expression }
```

### Example

```javascript
db.orders.aggregate([
  {
    $project: {
      discount: "$discountAmount",
      wholeDollarDiscount: { $floor: "$discountAmount" }
    }
  }
])
```

```javascript
{ discountAmount: 9.99 }  ->  { wholeDollarDiscount: 9 }
{ discountAmount: 5.01 }  ->  { wholeDollarDiscount: 5 }
{ discountAmount: 3.00 }  ->  { wholeDollarDiscount: 3 }
```

### Use Case - Bucketing by Grouping

```javascript
db.metrics.aggregate([
  {
    $addFields: {
      // Group values into buckets of 10
      bucket: { $multiply: [{ $floor: { $divide: ["$value", 10] } }, 10] }
    }
  },
  {
    $group: {
      _id: "$bucket",
      count: { $sum: 1 },
      avg: { $avg: "$value" }
    }
  },
  { $sort: { _id: 1 } }
])
```

## Combining $abs, $ceil, and $floor

```javascript
db.inventory.aggregate([
  {
    $addFields: {
      // Absolute stock difference
      stockDiff: { $abs: { $subtract: ["$currentStock", "$targetStock"] } },
      // Boxes needed to ship (round up to full boxes)
      boxesNeeded: {
        $ceil: { $divide: ["$quantityToShip", "$unitsPerBox"] }
      },
      // Discount in whole dollars
      appliedDiscount: { $floor: { $multiply: ["$price", "$discountRate"] } }
    }
  }
])
```

## $round - Related Operator

`$round` rounds to a specified number of decimal places:

```javascript
db.products.aggregate([
  {
    $project: {
      price: { $round: ["$rawPrice", 2] },      // 2 decimal places
      score: { $round: ["$rawScore", 0] },       // nearest integer
      thousands: { $round: ["$revenue", -3] }    // nearest 1000
    }
  }
])
```

## $trunc - Related Operator

`$trunc` truncates without rounding:

```javascript
db.sensors.aggregate([
  {
    $project: {
      reading: 1,
      truncated: { $trunc: ["$reading", 1] }  // keep 1 decimal, no rounding
    }
  }
])
```

```javascript
{ reading: 22.768 }  ->  { truncated: 22.7 }
{ reading: 22.999 }  ->  { truncated: 22.9 }
```

## Handling Null and Missing Values

These operators return `null` when the input is `null` or the field is missing:

```javascript
db.items.aggregate([
  {
    $project: {
      safeFloor: {
        $cond: {
          if: { $ifNull: ["$value", false] },
          then: { $floor: "$value" },
          else: 0
        }
      }
    }
  }
])
```

## Summary

The `$abs`, `$ceil`, and `$floor` operators handle common mathematical needs in MongoDB aggregation: `$abs` removes sign for magnitude comparisons, `$ceil` rounds up for conservative estimates like shipping and page counts, and `$floor` rounds down for conservative value reductions and bucketing. They integrate cleanly with other arithmetic operators and are null-safe by default.
