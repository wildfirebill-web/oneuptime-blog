# How to Use $pow and $sqrt and $log in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Math, $pow, $sqrt, $log, Pipeline

Description: Learn how to use MongoDB's $pow, $sqrt, and $log math operators in aggregation pipelines for exponential growth, distance calculations, and logarithmic transformations.

---

## Overview

MongoDB's aggregation framework includes advanced math operators for scientific and statistical computations. `$pow`, `$sqrt`, and `$log` enable exponential, root, and logarithmic calculations directly in the database, avoiding the need to retrieve raw data and process it in application code.

## Sample Data

```javascript
db.measurements.insertMany([
  { _id: 1, sensor: "A", value: 100, baseline: 10, area_sq_meters: 144 },
  { _id: 2, sensor: "B", value: 1000, baseline: 10, area_sq_meters: 625 },
  { _id: 3, sensor: "C", value: 50, baseline: 10, area_sq_meters: 36 },
  { _id: 4, sensor: "D", value: 0.5, baseline: 10, area_sq_meters: 2.25 }
]);
```

## Using $pow

`$pow` raises a number to a specified exponent.

### Syntax

```javascript
{ $pow: [ <base>, <exponent> ] }
```

### Example: Calculate Squared Values

```javascript
db.measurements.aggregate([
  {
    $project: {
      sensor: 1,
      value: 1,
      valueSquared: { $pow: ["$value", 2] }
    }
  }
]);
```

```text
{ sensor: "A", value: 100, valueSquared: 10000 }
{ sensor: "B", value: 1000, valueSquared: 1000000 }
{ sensor: "C", value: 50, valueSquared: 2500 }
{ sensor: "D", value: 0.5, valueSquared: 0.25 }
```

### Example: Compound Interest Calculation

```javascript
// Calculate future value: principal * (1 + rate)^years
db.investments.aggregate([
  {
    $project: {
      investorId: 1,
      futureValue: {
        $multiply: [
          "$principal",
          { $pow: [
            { $add: [1, "$annualRate"] },
            "$years"
          ]}
        ]
      }
    }
  }
]);
```

### Example: Euclidean Distance (3D)

```javascript
db.points.aggregate([
  {
    $project: {
      distanceFromOrigin: {
        $sqrt: {
          $add: [
            { $pow: ["$x", 2] },
            { $pow: ["$y", 2] },
            { $pow: ["$z", 2] }
          ]
        }
      }
    }
  }
]);
```

## Using $sqrt

`$sqrt` returns the square root of a non-negative number.

### Syntax

```javascript
{ $sqrt: <expression> }
```

### Example: Calculate Side Length from Area

```javascript
db.measurements.aggregate([
  {
    $project: {
      sensor: 1,
      area_sq_meters: 1,
      side_length: { $sqrt: "$area_sq_meters" }
    }
  }
]);
```

```text
{ sensor: "A", area_sq_meters: 144, side_length: 12 }
{ sensor: "B", area_sq_meters: 625, side_length: 25 }
{ sensor: "C", area_sq_meters: 36, side_length: 6 }
{ sensor: "D", area_sq_meters: 2.25, side_length: 1.5 }
```

### Example: Standard Deviation Approximation

```javascript
db.scores.aggregate([
  {
    $group: {
      _id: "$courseId",
      count: { $sum: 1 },
      sumScores: { $sum: "$score" },
      sumSquaredScores: { $sum: { $pow: ["$score", 2] } }
    }
  },
  {
    $project: {
      courseId: "$_id",
      mean: { $divide: ["$sumScores", "$count"] },
      stddev: {
        $sqrt: {
          $subtract: [
            { $divide: ["$sumSquaredScores", "$count"] },
            { $pow: [{ $divide: ["$sumScores", "$count"] }, 2] }
          ]
        }
      }
    }
  }
]);
```

## Using $log

`$log` returns the logarithm of a number in a specified base.

### Syntax

```javascript
{ $log: [ <number>, <base> ] }
```

### Example: Log Base 10 (Decibels, pH Scale)

```javascript
db.measurements.aggregate([
  {
    $project: {
      sensor: 1,
      value: 1,
      log10Value: { $log: ["$value", 10] }
    }
  }
]);
```

```text
{ sensor: "A", value: 100, log10Value: 2 }
{ sensor: "B", value: 1000, log10Value: 3 }
{ sensor: "C", value: 50, log10Value: 1.699 }
```

### Natural Log Using $ln

For natural logarithm (base e), MongoDB provides the dedicated `$ln` operator:

```javascript
db.measurements.aggregate([
  {
    $project: {
      sensor: 1,
      naturalLog: { $ln: "$value" }
    }
  }
]);
```

### Example: Log2 for Binary Calculations

```javascript
db.files.aggregate([
  {
    $project: {
      filename: 1,
      sizeBytes: 1,
      bitsRequired: {
        $ceil: { $log: ["$sizeBytes", 2] }
      }
    }
  }
]);
```

## Combining $pow, $sqrt, and $log

### Example: Signal-to-Noise Ratio in Decibels

```javascript
// SNR (dB) = 10 * log10(signal_power / noise_power)
db.signals.aggregate([
  {
    $project: {
      signalId: 1,
      snrDecibels: {
        $multiply: [
          10,
          { $log: [{ $divide: ["$signalPower", "$noisePower"] }, 10] }
        ]
      }
    }
  }
]);
```

### Example: Normalize Values to Log Scale for Bucketing

```javascript
db.measurements.aggregate([
  {
    $addFields: {
      logBucket: {
        $floor: { $log: ["$value", 10] }
      }
    }
  },
  {
    $group: {
      _id: "$logBucket",
      count: { $sum: 1 },
      sensors: { $push: "$sensor" }
    }
  },
  { $sort: { _id: 1 } }
]);
```

```text
{ _id: -1, count: 1, sensors: ["D"] }   // 0.1 to 1
{ _id: 1, count: 1, sensors: ["C"] }    // 10 to 99
{ _id: 2, count: 1, sensors: ["A"] }    // 100 to 999
{ _id: 3, count: 1, sensors: ["B"] }    // 1000 to 9999
```

## Handling Edge Cases

`$sqrt` on a negative number returns an error. `$log` on zero or negative numbers also returns an error. Guard against these:

```javascript
db.measurements.aggregate([
  {
    $project: {
      sensor: 1,
      safeSqrt: {
        $cond: {
          if: { $gte: ["$value", 0] },
          then: { $sqrt: "$value" },
          else: null
        }
      },
      safeLog: {
        $cond: {
          if: { $gt: ["$value", 0] },
          then: { $log: ["$value", 10] },
          else: null
        }
      }
    }
  }
]);
```

## Summary

MongoDB's `$pow`, `$sqrt`, and `$log` operators bring scientific computing capabilities to aggregation pipelines. Use `$pow` for exponential calculations like compound interest or distance formulas, `$sqrt` for root-based computations like standard deviation or Euclidean distance, and `$log` or `$ln` for logarithmic transformations useful in decibel calculations, log-scale bucketing, and normalization. These operators compose freely with other arithmetic and conditional expressions.
