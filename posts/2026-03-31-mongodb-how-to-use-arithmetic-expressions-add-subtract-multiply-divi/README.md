# How to Use Arithmetic Expressions in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Arithmetic Expressions, $add, $subtract, $multiply, $divide, NoSQL

Description: Learn how to use MongoDB's arithmetic aggregation expressions $add, $subtract, $multiply, and $divide to compute values within aggregation pipelines.

---

## Overview of Arithmetic Expressions

MongoDB provides a rich set of arithmetic expression operators for use within aggregation pipelines. These operators work in `$project`, `$addFields`, `$group`, `$match` (with `$expr`), and other stages.

The four fundamental operators are:
- `$add` - adds numbers or dates
- `$subtract` - subtracts numbers or dates
- `$multiply` - multiplies numbers
- `$divide` - divides numbers

## $add

Adds two or more numbers together. When one operand is a date, `$add` treats the numeric operand as milliseconds.

```javascript
// Syntax: { $add: [expr1, expr2, ...] }

db.orders.aggregate([
  {
    $project: {
      subtotal: "$price",
      tax: { $multiply: ["$price", 0.08] },
      total: { $add: ["$price", { $multiply: ["$price", 0.08] }] }
    }
  }
])
```

Adding to a date (milliseconds):

```javascript
db.subscriptions.aggregate([
  {
    $addFields: {
      // Add 30 days (in ms) to start date
      expiresAt: { $add: ["$startDate", 30 * 24 * 60 * 60 * 1000] }
    }
  }
])
```

## $subtract

Subtracts the second value from the first. When both operands are dates, returns the difference in milliseconds.

```javascript
// Subtract a discount from a price
db.products.aggregate([
  {
    $project: {
      name: 1,
      originalPrice: "$price",
      discountedPrice: { $subtract: ["$price", "$discount"] }
    }
  }
])
```

Calculating time elapsed between dates:

```javascript
db.sessions.aggregate([
  {
    $project: {
      userId: 1,
      durationMs: { $subtract: ["$endTime", "$startTime"] },
      durationMinutes: {
        $divide: [{ $subtract: ["$endTime", "$startTime"] }, 60000]
      }
    }
  }
])
```

## $multiply

Multiplies two or more numbers together.

```javascript
// Syntax: { $multiply: [expr1, expr2, ...] }

db.orders.aggregate([
  {
    $project: {
      item: 1,
      lineTotal: { $multiply: ["$price", "$quantity"] },
      lineWithTax: { $multiply: ["$price", "$quantity", 1.08] }
    }
  }
])
```

Multiple factors:

```javascript
db.inventory.aggregate([
  {
    $addFields: {
      totalValue: { $multiply: ["$unitPrice", "$quantity", "$condition_multiplier"] }
    }
  }
])
```

## $divide

Divides the first number by the second.

```javascript
// Syntax: { $divide: [dividend, divisor] }

db.products.aggregate([
  {
    $project: {
      name: 1,
      pricePerUnit: { $divide: ["$bulkPrice", "$unitsPerBulk"] },
      marginPercent: {
        $divide: [
          { $subtract: ["$price", "$cost"] },
          "$price"
        ]
      }
    }
  }
])
```

## Combining Operators

Arithmetic operators can be nested:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      // profit = (price - cost) * quantity
      profit: {
        $multiply: [
          { $subtract: ["$price", "$cost"] },
          "$quantity"
        ]
      },
      // profitMargin = (price - cost) / price * 100
      profitMarginPct: {
        $multiply: [
          { $divide: [
            { $subtract: ["$price", "$cost"] },
            "$price"
          ]},
          100
        ]
      }
    }
  }
])
```

## Using with $group

Compute running totals and averages:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: "$region",
      totalRevenue: { $sum: { $multiply: ["$price", "$quantity"] } },
      totalCost: { $sum: { $multiply: ["$cost", "$quantity"] } },
      totalOrders: { $sum: 1 }
    }
  },
  {
    $addFields: {
      grossProfit: { $subtract: ["$totalRevenue", "$totalCost"] },
      avgOrderValue: { $divide: ["$totalRevenue", "$totalOrders"] }
    }
  }
])
```

## Using with $match via $expr

Filter based on computed arithmetic conditions:

```javascript
// Find products where profit margin exceeds 30%
db.products.aggregate([
  {
    $match: {
      $expr: {
        $gt: [
          { $divide: [
            { $subtract: ["$price", "$cost"] },
            "$price"
          ]},
          0.30
        ]
      }
    }
  }
])
```

## Handling Division by Zero

Use `$cond` to guard against division by zero:

```javascript
db.products.aggregate([
  {
    $project: {
      margin: {
        $cond: {
          if: { $eq: ["$price", 0] },
          then: null,
          else: { $divide: [{ $subtract: ["$price", "$cost"] }, "$price"] }
        }
      }
    }
  }
])
```

## Summary

MongoDB's arithmetic expression operators - `$add`, `$subtract`, `$multiply`, and `$divide` - enable rich in-pipeline calculations for computed fields, aggregated metrics, and conditional filtering. They can be freely nested and combined, and work across `$project`, `$addFields`, `$group`, and `$match` stages. Always guard against division by zero with `$cond` for robust pipelines.
