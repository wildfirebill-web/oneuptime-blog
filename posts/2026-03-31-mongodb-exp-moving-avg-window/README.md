# How to Use $expMovingAvg for Smoothed Trends in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Window Function, Aggregation, Exponential Moving Average, Time Series

Description: Learn how to use $expMovingAvg in MongoDB's $setWindowFields to compute exponentially weighted moving averages for smoothing trend data.

---

The `$expMovingAvg` (Exponential Moving Average) window function gives more weight to recent values and progressively less weight to older ones. Unlike a simple moving average that weights all points equally, EMA reacts faster to recent changes while still smoothing out short-term noise - making it valuable for financial analysis, monitoring dashboards, and trend detection.

## How $expMovingAvg Works

`$expMovingAvg` can be configured in two ways:

1. **`N` parameter** - specifies the number of historical points to emphasize (alpha = 2/(N+1))
2. **`alpha` parameter** - directly specifies the smoothing factor between 0 and 1

Higher alpha = more weight on recent values (faster response, less smoothing).
Lower alpha = more weight on historical values (slower response, more smoothing).

## Basic EMA with N Parameter

Compute a 10-period EMA of daily closing prices per stock ticker:

```javascript
db.stock_prices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { tradingDate: 1 },
      output: {
        ema10: {
          $expMovingAvg: {
            input: "$closePrice",
            N: 10
          }
        }
      }
    }
  },
  {
    $project: {
      ticker: 1,
      tradingDate: 1,
      closePrice: 1,
      ema10: { $round: ["$ema10", 2] }
    }
  }
]);
```

The EMA10 reacts to price changes within 10 trading periods, smoothing out daily noise.

## Using the Alpha Parameter for Custom Smoothing

When you need precise control over the smoothing factor, use `alpha` directly:

```javascript
db.server_metrics.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$serverId",
      sortBy: { timestamp: 1 },
      output: {
        smoothedCpuUsage: {
          $expMovingAvg: {
            input: "$cpuPercent",
            alpha: 0.3    // 30% weight on most recent value
          }
        }
      }
    }
  }
]);
```

With `alpha: 0.3`, the current value contributes 30% to the EMA, while historical values contribute 70%.

## Comparing Fast and Slow EMAs for Trend Detection

Using two EMAs with different periods is a common trading signal technique. Apply it to any time-series metric:

```javascript
db.daily_sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$productId",
      sortBy: { saleDate: 1 },
      output: {
        ema5: {
          $expMovingAvg: { input: "$revenue", N: 5 }
        },
        ema20: {
          $expMovingAvg: { input: "$revenue", N: 20 }
        }
      }
    }
  },
  {
    $addFields: {
      trend: {
        $cond: [
          { $gt: ["$ema5", "$ema20"] },
          "uptrend",
          { $cond: [
            { $lt: ["$ema5", "$ema20"] },
            "downtrend",
            "neutral"
          ]}
        ]
      }
    }
  }
]);
```

When the 5-period EMA crosses above the 20-period EMA, it signals an uptrend in revenue.

## EMA for Anomaly Detection

Use EMA as a baseline and flag readings that deviate significantly:

```javascript
db.sensor_readings.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$sensorId",
      sortBy: { readingTime: 1 },
      output: {
        expectedValue: {
          $expMovingAvg: { input: "$reading", N: 20 }
        }
      }
    }
  },
  {
    $addFields: {
      deviation: { $abs: { $subtract: ["$reading", "$expectedValue"] } },
      deviationPercent: {
        $multiply: [
          {
            $divide: [
              { $abs: { $subtract: ["$reading", "$expectedValue"] } },
              "$expectedValue"
            ]
          },
          100
        ]
      }
    }
  },
  { $match: { deviationPercent: { $gt: 25 } } }
]);
```

## Summary

`$expMovingAvg` is the right window function when you need a responsive moving average that weighs recent data more heavily. Use the `N` parameter for period-based smoothing aligned with domain conventions, or `alpha` for precise control. Combining fast and slow EMAs enables trend detection, and comparing current values to their EMA reveals anomalies - making `$expMovingAvg` a versatile tool for time-series analytics in MongoDB.
