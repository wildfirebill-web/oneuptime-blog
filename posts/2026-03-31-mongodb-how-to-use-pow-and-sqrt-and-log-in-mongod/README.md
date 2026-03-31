# How to Use $pow, $sqrt, and $log in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Math Expressions, Database

Description: Learn how to use $pow, $sqrt, and $log math expressions in MongoDB aggregation pipelines for numerical calculations and data transformations.

---

## Overview

MongoDB aggregation pipelines support a rich set of arithmetic operators that let you perform mathematical computations directly in your queries. The `$pow`, `$sqrt`, and `$log` operators cover exponentiation, square roots, and logarithmic calculations - all without needing post-processing in application code.

## Using $pow to Calculate Powers

The `$pow` operator raises a number to the specified exponent. It takes two arguments: the base and the exponent.

```javascript
db.measurements.aggregate([
  {
    $project: {
      value: 1,
      squared: { $pow: ["$value", 2] },
      cubed: { $pow: ["$value", 3] },
      squareRoot: { $pow: ["$value", 0.5] }
    }
  }
])
```

A practical example: computing the area of a circle from radius stored in documents.

```javascript
db.circles.aggregate([
  {
    $project: {
      radius: 1,
      area: {
        $multiply: [
          3.14159,
          { $pow: ["$radius", 2] }
        ]
      }
    }
  }
])
```

## Using $sqrt to Calculate Square Roots

The `$sqrt` operator returns the square root of a positive number. It accepts a single numeric expression.

```javascript
db.triangles.aggregate([
  {
    $project: {
      sideA: 1,
      sideB: 1,
      hypotenuse: {
        $sqrt: {
          $add: [
            { $pow: ["$sideA", 2] },
            { $pow: ["$sideB", 2] }
          ]
        }
      }
    }
  }
])
```

This uses the Pythagorean theorem inline in the aggregation pipeline. `$sqrt` returns `null` if the input is null or missing, and raises an error for negative numbers.

## Using $log for Logarithmic Calculations

The `$log` operator computes the logarithm of a number in a specified base. It takes the number and the base as arguments.

```javascript
db.metrics.aggregate([
  {
    $project: {
      value: 1,
      log10: { $log: ["$value", 10] },
      log2: { $log: ["$value", 2] },
      naturalLog: { $ln: "$value" }
    }
  }
])
```

Note: For natural logarithm (base e), MongoDB provides the dedicated `$ln` operator. For log base 10, you can also use `$log10`.

## Combining Math Operators in Complex Calculations

These operators can be combined to build complex formulas. The following example computes a normalized score using logarithmic scaling:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      price: 1,
      views: 1,
      popularityScore: {
        $multiply: [
          { $log: [{ $add: ["$views", 1] }, 10] },
          { $pow: [{ $divide: [1, "$price"] }, 0.5] }
        ]
      }
    }
  },
  { $sort: { popularityScore: -1 } },
  { $limit: 10 }
])
```

## Handling Edge Cases

When working with these operators, keep these behaviors in mind:

- `$pow` with exponent 0 always returns 1
- `$sqrt` of 0 returns 0; negative inputs cause an error
- `$log` requires positive numbers for both the value and the base
- All three operators return `null` if any input is null or missing

```javascript
db.data.aggregate([
  {
    $project: {
      safeLog: {
        $cond: {
          if: { $gt: ["$value", 0] },
          then: { $log: ["$value", 10] },
          else: null
        }
      }
    }
  }
])
```

## Summary

The `$pow`, `$sqrt`, and `$log` operators in MongoDB aggregation allow you to perform mathematical calculations directly in your pipeline stages without application-side processing. These operators integrate seamlessly with other aggregation expressions, enabling complex formulas for analytics, scoring, and data normalization. Always guard against null or invalid inputs using `$cond` or `$ifNull` to keep pipelines robust.
