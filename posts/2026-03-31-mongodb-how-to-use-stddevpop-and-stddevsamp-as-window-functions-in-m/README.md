# How to Use $stdDevPop and $stdDevSamp as Window Functions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Window Function, Statistics, Aggregation, Analytics

Description: Learn how to use $stdDevPop and $stdDevSamp as window functions in MongoDB to compute rolling and partition-level standard deviations for statistical analysis.

---

## Standard Deviation as a Window Function

MongoDB's `$setWindowFields` stage (5.0+) supports `$stdDevPop` and `$stdDevSamp` as window operators. These compute standard deviation over a sliding or expanding window without collapsing rows, making them ideal for anomaly detection, financial volatility analysis, and quality control.

- `$stdDevPop` - population standard deviation (use when your window covers the entire population)
- `$stdDevSamp` - sample standard deviation (use when your window is a sample; divides by N-1)

## Setup - Sample Dataset

```javascript
db.sensorReadings.insertMany([
  { sensor: "A", hour: 1, temperature: 22.1 },
  { sensor: "A", hour: 2, temperature: 22.5 },
  { sensor: "A", hour: 3, temperature: 23.0 },
  { sensor: "A", hour: 4, temperature: 45.8 },  // anomaly spike
  { sensor: "A", hour: 5, temperature: 22.3 },
  { sensor: "A", hour: 6, temperature: 22.7 },
  { sensor: "B", hour: 1, temperature: 18.0 },
  { sensor: "B", hour: 2, temperature: 18.5 },
  { sensor: "B", hour: 3, temperature: 19.1 },
  { sensor: "B", hour: 4, temperature: 18.8 },
  { sensor: "B", hour: 5, temperature: 18.3 },
  { sensor: "B", hour: 6, temperature: 18.6 }
]);
```

## Example 1 - Partition-Wide Standard Deviation

Compute standard deviation across the entire partition (all hours per sensor):

```javascript
db.sensorReadings.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$sensor",
      sortBy: { hour: 1 },
      output: {
        stdDevPopAll: {
          $stdDevPop: "$temperature",
          window: { documents: ["unbounded", "unbounded"] }
        },
        stdDevSampAll: {
          $stdDevSamp: "$temperature",
          window: { documents: ["unbounded", "unbounded"] }
        },
        avgTemp: {
          $avg: "$temperature",
          window: { documents: ["unbounded", "unbounded"] }
        }
      }
    }
  }
]);
```

## Example 2 - Rolling Standard Deviation (Sliding Window)

Compute standard deviation over a rolling 3-hour window:

```javascript
db.sensorReadings.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$sensor",
      sortBy: { hour: 1 },
      output: {
        rollingAvg: {
          $avg: "$temperature",
          window: { documents: [-2, 0] }  // current + 2 prior = 3 rows
        },
        rollingStdDevPop: {
          $stdDevPop: "$temperature",
          window: { documents: [-2, 0] }
        }
      }
    }
  },
  { $sort: { sensor: 1, hour: 1 } }
]);
```

```text
sensor: A, hour: 1, temp: 22.1, rollingAvg: 22.1, rollingStdDevPop: 0
sensor: A, hour: 2, temp: 22.5, rollingAvg: 22.3, rollingStdDevPop: 0.2
sensor: A, hour: 3, temp: 23.0, rollingAvg: 22.5, rollingStdDevPop: 0.37
sensor: A, hour: 4, temp: 45.8, rollingAvg: 30.4, rollingStdDevPop: 10.8  <- spike
```

## Example 3 - Anomaly Detection Using Z-Score

Flag readings more than 2 standard deviations from the partition mean:

```javascript
db.sensorReadings.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$sensor",
      sortBy: { hour: 1 },
      output: {
        partitionAvg: {
          $avg: "$temperature",
          window: { documents: ["unbounded", "unbounded"] }
        },
        partitionStdDev: {
          $stdDevPop: "$temperature",
          window: { documents: ["unbounded", "unbounded"] }
        }
      }
    }
  },
  {
    $addFields: {
      zScore: {
        $cond: {
          if: { $eq: ["$partitionStdDev", 0] },
          then: 0,
          else: {
            $divide: [
              { $abs: { $subtract: ["$temperature", "$partitionAvg"] } },
              "$partitionStdDev"
            ]
          }
        }
      }
    }
  },
  {
    $addFields: {
      isAnomaly: { $gt: ["$zScore", 2] }
    }
  },
  { $sort: { sensor: 1, hour: 1 } }
]);
```

## Example 4 - Expanding Window Standard Deviation

Use an expanding window (from start of partition to current row) to see how volatility grows as more data is added:

```javascript
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { tradingDay: 1 },
      output: {
        expandingStdDev: {
          $stdDevSamp: "$closingPrice",
          window: { documents: ["unbounded", "current"] }
        },
        expandingAvg: {
          $avg: "$closingPrice",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
]);
```

## Example 5 - Coefficient of Variation

Combine `$stdDevPop` with `$avg` to compute coefficient of variation (relative variability):

```javascript
db.products.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { date: 1 },
      output: {
        avgSales: {
          $avg: "$dailySales",
          window: { documents: ["unbounded", "unbounded"] }
        },
        stdDevSales: {
          $stdDevPop: "$dailySales",
          window: { documents: ["unbounded", "unbounded"] }
        }
      }
    }
  },
  {
    $addFields: {
      coefficientOfVariation: {
        $cond: {
          if: { $eq: ["$avgSales", 0] },
          then: null,
          else: { $divide: ["$stdDevSales", "$avgSales"] }
        }
      }
    }
  }
]);
```

## When to Use $stdDevPop vs $stdDevSamp

```text
Use $stdDevPop when:
- Your window contains the complete population you care about
- Analyzing all readings from a specific sensor/device
- Final reporting on a closed dataset

Use $stdDevSamp when:
- Your window is a sample from a larger population
- Rolling windows (only have a subset of future data)
- Estimating population variability from recent observations
```

## Summary

`$stdDevPop` and `$stdDevSamp` as window functions in MongoDB's `$setWindowFields` enable partition-level and rolling statistical deviation calculations without losing individual rows. They are most powerful for anomaly detection by computing per-row z-scores against a partition mean, for financial volatility analysis with expanding or sliding windows, and for quality control dashboards comparing current variance against historical norms. Always choose `$stdDevSamp` for rolling windows and `$stdDevPop` when your window represents the complete dataset of interest.
