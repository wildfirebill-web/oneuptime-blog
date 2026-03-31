# How to Perform Statistical Analysis with MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Statistics, Analytics, Pipeline

Description: Learn how to perform statistical analysis directly in MongoDB using aggregation operators for mean, median, standard deviation, percentiles, and correlation calculations.

---

MongoDB's aggregation pipeline includes statistical operators that let you compute distributions, standard deviations, and percentiles without exporting data to a separate analytics tool.

## Basic Descriptive Statistics

Compute mean, min, max, and standard deviation in a single pipeline:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: null,
      count: { $sum: 1 },
      mean: { $avg: "$amount" },
      min: { $min: "$amount" },
      max: { $max: "$amount" },
      stdDev: { $stdDevSamp: "$amount" },
      stdDevPop: { $stdDevPop: "$amount" }
    }
  }
])
```

Use `$stdDevSamp` for a sample (n-1 denominator) or `$stdDevPop` for the full population.

## Percentiles (MongoDB 7.0+)

Calculate the median (50th percentile) and other quantiles:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: "$category",
      median: {
        $percentile: { input: "$amount", p: [0.5], method: "approximate" }
      },
      p25: {
        $percentile: { input: "$amount", p: [0.25], method: "approximate" }
      },
      p75: {
        $percentile: { input: "$amount", p: [0.75], method: "approximate" }
      },
      p95: {
        $percentile: { input: "$amount", p: [0.95], method: "approximate" }
      }
    }
  }
])
```

The result of `$percentile` is an array; index 0 corresponds to the first `p` value.

## Frequency Distribution (Histogram Buckets)

Use `$bucket` to create a histogram of order amounts:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $bucket: {
      groupBy: "$amount",
      boundaries: [0, 25, 50, 100, 200, 500, 1000],
      default: "1000+",
      output: {
        count: { $sum: 1 },
        totalRevenue: { $sum: "$amount" }
      }
    }
  }
])
```

## Z-Score Calculation

Identify statistical outliers by computing z-scores:

```javascript
// Step 1: Compute mean and stddev
const stats = db.orders.aggregate([
  { $group: { _id: null, mean: { $avg: "$amount" }, stddev: { $stdDevSamp: "$amount" } } }
]).toArray()[0]

// Step 2: Compute z-score per document
db.orders.aggregate([
  {
    $addFields: {
      zScore: {
        $divide: [
          { $subtract: ["$amount", stats.mean] },
          stats.stddev
        ]
      }
    }
  },
  { $match: { zScore: { $gt: 3 } } }  // Outliers beyond 3 standard deviations
])
```

## Moving Average with $setWindowFields

Calculate a 7-day moving average for time series data:

```javascript
db.daily_sales.aggregate([
  { $sort: { date: 1 } },
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        movingAvg7d: {
          $avg: "$revenue",
          window: { range: [-6, 0], unit: "day" }
        },
        rollingStdDev: {
          $stdDevSamp: "$revenue",
          window: { range: [-6, 0], unit: "day" }
        }
      }
    }
  }
])
```

## Correlation Between Two Fields

Compute Pearson correlation using aggregation math operators:

```javascript
db.orders.aggregate([
  { $group: {
      _id: null,
      n: { $sum: 1 },
      sumX: { $sum: "$quantity" },
      sumY: { $sum: "$amount" },
      sumXY: { $sum: { $multiply: ["$quantity", "$amount"] } },
      sumX2: { $sum: { $multiply: ["$quantity", "$quantity"] } },
      sumY2: { $sum: { $multiply: ["$amount", "$amount"] } }
  }},
  { $addFields: {
      correlation: {
        $divide: [
          { $subtract: [{ $multiply: ["$n", "$sumXY"] }, { $multiply: ["$sumX", "$sumY"] }] },
          { $sqrt: {
            $multiply: [
              { $subtract: [{ $multiply: ["$n", "$sumX2"] }, { $multiply: ["$sumX", "$sumX"] }] },
              { $subtract: [{ $multiply: ["$n", "$sumY2"] }, { $multiply: ["$sumY", "$sumY"] }] }
            ]
          }}
        ]
      }
  }}
])
```

## Summary

MongoDB's aggregation pipeline provides a rich set of statistical operators including `$stdDevSamp`, `$percentile`, `$bucket` for histograms, and `$setWindowFields` for moving statistics. For more complex analyses like z-scores and correlation, combine multiple pipeline stages and arithmetic operators. These tools let you perform most common statistical analyses without leaving MongoDB.
