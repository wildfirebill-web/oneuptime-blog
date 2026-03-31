# How to Use $add, $subtract, $multiply, $divide in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Expression, Arithmetic

Description: A practical guide to MongoDB's four core arithmetic expressions - $add, $subtract, $multiply, and $divide - with real-world examples for computing derived fields.

---

## Overview of Arithmetic Expressions

MongoDB's aggregation pipeline supports four fundamental arithmetic operators that compute numeric values from field references and constants: `$add`, `$subtract`, `$multiply`, and `$divide`. These expressions are available wherever an expression is accepted - in `$project`, `$addFields`, `$group`, `$match` with `$expr`, and conditional operators.

## $add

Adds two or more numbers. When one argument is a Date, it adds milliseconds and returns a new Date.

```javascript
// Sum two fields
{ $add: ["$price", "$tax"] }

// Sum three fields
{ $add: ["$base", "$fee", "$adjustment"] }

// Add constant
{ $add: ["$score", 10] }

// Add 7 days to a date
{ $add: ["$createdAt", 7 * 24 * 60 * 60 * 1000] }
```

## $subtract

Subtracts the second argument from the first. When both arguments are Dates, returns the difference in milliseconds.

```javascript
// Subtract two numbers
{ $subtract: ["$revenue", "$cost"] }

// Compute duration in minutes
{
  $divide: [
    { $subtract: ["$endTime", "$startTime"] },
    60000
  ]
}
```

## $multiply

Multiplies two or more numbers.

```javascript
// Price times quantity
{ $multiply: ["$unitPrice", "$quantity"] }

// Apply a discount rate
{ $multiply: ["$price", 0.85] }

// Multi-factor computation
{ $multiply: ["$rate", "$hours", "$difficulty"] }
```

## $divide

Divides the first argument by the second.

```javascript
// Convert bytes to megabytes
{ $divide: ["$sizeBytes", 1048576] }

// Compute average
{ $divide: ["$totalScore", "$count"] }
```

Always guard against division by zero with `$cond`:

```javascript
{
  $cond: {
    if: { $eq: ["$count", 0] },
    then: null,
    else: { $divide: ["$totalScore", "$count"] }
  }
}
```

## Practical Example: Invoice Line Items

```javascript
db.invoices.aggregate([
  {
    $project: {
      invoiceId: 1,
      items: 1,
      subtotal: {
        $sum: {
          $map: {
            input: "$items",
            as: "item",
            in: { $multiply: ["$$item.qty", "$$item.unitPrice"] }
          }
        }
      }
    }
  },
  {
    $addFields: {
      tax: { $multiply: ["$subtotal", 0.08] },
      total: { $multiply: ["$subtotal", 1.08] }
    }
  }
])
```

## Using Arithmetic in $group

```javascript
db.timeEntries.aggregate([
  {
    $group: {
      _id: "$projectId",
      totalHours: { $sum: "$hours" },
      laborCost: {
        $sum: { $multiply: ["$hours", "$hourlyRate"] }
      }
    }
  },
  {
    $addFields: {
      avgHourlyRate: {
        $cond: {
          if: { $eq: ["$totalHours", 0] },
          then: 0,
          else: { $divide: ["$laborCost", "$totalHours"] }
        }
      }
    }
  }
])
```

## Filtering with Arithmetic via $expr

```javascript
db.products.aggregate([
  {
    $match: {
      $expr: {
        $gte: [
          { $subtract: ["$salePrice", "$costPrice"] },
          20
        ]
      }
    }
  }
])
// Products with margin >= $20
```

## Nesting Multiple Operators

```javascript
// Compound interest calculation
db.accounts.aggregate([
  {
    $project: {
      futureValue: {
        $multiply: [
          "$principal",
          { $pow: [{ $add: [1, "$rate"] }, "$years"] }
        ]
      }
    }
  }
])
```

## Summary

The four arithmetic expressions `$add`, `$subtract`, `$multiply`, and `$divide` are the building blocks for numeric computation in MongoDB aggregation. They nest freely, integrate with all expression-accepting stages, and eliminate the need to fetch documents to the application layer just to compute derived values. Always protect `$divide` operations with a zero check using `$cond`.
