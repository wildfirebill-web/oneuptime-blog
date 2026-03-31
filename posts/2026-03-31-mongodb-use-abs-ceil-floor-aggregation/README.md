# How to Use $abs and $ceil and $floor in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Expression, Arithmetic

Description: Learn how to use MongoDB's $abs, $ceil, and $floor aggregation expressions to compute absolute values, round up, and round down numbers within aggregation pipelines.

---

## Overview

MongoDB's aggregation framework includes several math expressions for working with numeric precision: `$abs` returns the absolute value of a number, `$ceil` rounds a number up to the nearest integer, and `$floor` rounds a number down to the nearest integer. These are available in any aggregation stage that accepts expressions.

## $abs: Absolute Value

The `$abs` expression returns the absolute (non-negative) value of a number.

```javascript
// Basic usage
{ $abs: "$value" }
{ $abs: -15 }   // Returns 15
{ $abs: 15 }    // Returns 15
```

### Example: Computing Absolute Difference

```javascript
db.measurements.aggregate([
  {
    $project: {
      deviation: {
        $abs: { $subtract: ["$actual", "$expected"] }
      }
    }
  }
])
// Returns how far actual is from expected, regardless of direction
```

### Example: Ranking by Distance from Target

```javascript
db.products.aggregate([
  {
    $addFields: {
      priceDistance: { $abs: { $subtract: ["$price", targetPrice] } }
    }
  },
  { $sort: { priceDistance: 1 } },
  { $limit: 10 }
])
// Products closest to the target price
```

## $ceil: Round Up

The `$ceil` expression returns the smallest integer greater than or equal to the specified number.

```javascript
{ $ceil: 4.1 }   // Returns 5
{ $ceil: 4.9 }   // Returns 5
{ $ceil: 4.0 }   // Returns 4
{ $ceil: -4.1 }  // Returns -4  (rounds toward zero for negatives)
```

### Example: Computing Required Pages

```javascript
db.reports.aggregate([
  {
    $project: {
      totalItems: 1,
      pagesNeeded: {
        $ceil: { $divide: ["$totalItems", 25] }
      }
    }
  }
])
// 26 items -> ceil(26/25) = ceil(1.04) = 2 pages
```

### Example: Ceiling a Rate

```javascript
db.invoices.aggregate([
  {
    $project: {
      hoursWorked: 1,
      billedHours: { $ceil: "$hoursWorked" }
    }
  }
])
// Round up partial hours for billing
```

## $floor: Round Down

The `$floor` expression returns the largest integer less than or equal to the specified number.

```javascript
{ $floor: 4.9 }   // Returns 4
{ $floor: 4.1 }   // Returns 4
{ $floor: 4.0 }   // Returns 4
{ $floor: -4.1 }  // Returns -5  (rounds away from zero for negatives)
```

### Example: Age from Birthdate

```javascript
db.users.aggregate([
  {
    $project: {
      age: {
        $floor: {
          $divide: [
            { $subtract: [new Date(), "$birthDate"] },
            365.25 * 24 * 60 * 60 * 1000  // ms per year
          ]
        }
      }
    }
  }
])
```

### Example: Binning Values into Decades

```javascript
db.products.aggregate([
  {
    $project: {
      priceBucket: {
        $multiply: [
          { $floor: { $divide: ["$price", 10] } },
          10
        ]
      }
    }
  }
])
// $0-$9 -> bucket 0, $10-$19 -> bucket 10, etc.
```

## Combining All Three

```javascript
db.experiments.aggregate([
  {
    $project: {
      errorAbs: { $abs: { $subtract: ["$measured", "$reference"] } },
      errorCeiled: {
        $ceil: { $abs: { $subtract: ["$measured", "$reference"] } }
      },
      errorFloored: {
        $floor: { $abs: { $subtract: ["$measured", "$reference"] } }
      }
    }
  }
])
```

## $round as a Related Operator

For rounding to a specific number of decimal places, use `$round`.

```javascript
{ $round: ["$value", 2] }  // Round to 2 decimal places
{ $round: ["$value", 0] }  // Round to nearest integer (half-to-even)
```

Unlike `$ceil` and `$floor` which always round in one direction, `$round` uses banker's rounding (half-to-even).

## Null and Missing Handling

All three operators return `null` when the input is `null` or when the referenced field is missing from the document.

```javascript
{ $ceil: null }     // Returns null
{ $floor: "$missingField" }  // Returns null
{ $abs: null }      // Returns null
```

## Summary

The `$abs`, `$ceil`, and `$floor` expressions provide precise numeric control in MongoDB aggregation pipelines. Use `$abs` for magnitude calculations and error analysis, `$ceil` for rounding up (billing hours, page counts), and `$floor` for rounding down (age calculations, bucket assignments). All three return `null` gracefully when their input is missing or null.
