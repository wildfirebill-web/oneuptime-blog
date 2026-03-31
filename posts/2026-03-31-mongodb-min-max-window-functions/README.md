# How to Use $min and $max as Window Functions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Window Function, Aggregation, Analytics, Min Max

Description: Learn how to use $min and $max as window functions in MongoDB's $setWindowFields to find running extremes and sliding window peaks over ordered data.

---

The `$min` and `$max` window functions in MongoDB's `$setWindowFields` stage compute minimum and maximum values over a sliding window of documents. These are essential for time-series analysis, finding running extremes, and detecting anomalies relative to recent data ranges.

## Running Minimum and Maximum

Track the running minimum and maximum price ever seen for each product:

```javascript
db.price_history.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$productId",
      sortBy: { recordedAt: 1 },
      output: {
        runningMin: {
          $min: "$price",
          window: { documents: ["unbounded", "current"] }
        },
        runningMax: {
          $max: "$price",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  },
  {
    $project: {
      productId: 1,
      recordedAt: 1,
      price: 1,
      allTimeMin: "$runningMin",
      allTimeMax: "$runningMax",
      percentFromHigh: {
        $multiply: [
          { $divide: [{ $subtract: ["$price", "$runningMax"] }, "$runningMax"] },
          100
        ]
      }
    }
  }
]);
```

## Sliding Window High and Low (30-Day Range)

Find the 30-day high and low for stock prices:

```javascript
db.stock_prices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { tradingDate: 1 },
      output: {
        high30Day: {
          $max: "$closePrice",
          window: {
            range: [-29, 0],
            unit: "day"
          }
        },
        low30Day: {
          $min: "$closePrice",
          window: {
            range: [-29, 0],
            unit: "day"
          }
        }
      }
    }
  },
  {
    $addFields: {
      range30Day: { $subtract: ["$high30Day", "$low30Day"] },
      positionInRange: {
        $cond: [
          { $eq: [{ $subtract: ["$high30Day", "$low30Day"] }, 0] },
          0,
          {
            $divide: [
              { $subtract: ["$closePrice", "$low30Day"] },
              { $subtract: ["$high30Day", "$low30Day"] }
            ]
          }
        ]
      }
    }
  }
]);
```

`positionInRange` of 1.0 means price is at 30-day high; 0.0 means at 30-day low.

## Partition Min and Max for Normalization

Compute partition-wide min and max to normalize values to a 0-1 scale:

```javascript
db.sensor_readings.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$sensorId",
      sortBy: { recordedAt: 1 },
      output: {
        partitionMin: {
          $min: "$value",
          window: { documents: ["unbounded", "unbounded"] }
        },
        partitionMax: {
          $max: "$value",
          window: { documents: ["unbounded", "unbounded"] }
        }
      }
    }
  },
  {
    $addFields: {
      normalizedValue: {
        $cond: [
          { $eq: ["$partitionMin", "$partitionMax"] },
          0,
          {
            $divide: [
              { $subtract: ["$value", "$partitionMin"] },
              { $subtract: ["$partitionMax", "$partitionMin"] }
            ]
          }
        ]
      }
    }
  }
]);
```

## Detecting Anomalies Using Window Range

Flag readings that are far outside the recent window range:

```javascript
db.metrics.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$metricName",
      sortBy: { timestamp: 1 },
      output: {
        recentMin: {
          $min: "$value",
          window: { range: [-60, -1], unit: "minute" }
        },
        recentMax: {
          $max: "$value",
          window: { range: [-60, -1], unit: "minute" }
        }
      }
    }
  },
  {
    $addFields: {
      isAnomaly: {
        $or: [
          { $lt: ["$value", { $multiply: ["$recentMin", 0.5] }] },
          { $gt: ["$value", { $multiply: ["$recentMax", 2.0] }] }
        ]
      }
    }
  },
  { $match: { isAnomaly: true } }
]);
```

## Summary

The `$min` and `$max` window functions enable running extremes tracking, sliding window high/low calculations, data normalization, and anomaly detection in MongoDB aggregation pipelines. Use range-based windows with `unit` for time-series data, document-count windows for ordered sequences, and `["unbounded", "unbounded"]` for partition-wide extremes needed in normalization formulas.
