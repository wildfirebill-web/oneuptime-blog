# How to Use $expMovingAvg for Exponential Moving Averages in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Moving Average, Analytics

Description: Learn how to compute exponential moving averages in MongoDB using the $expMovingAvg operator inside $setWindowFields for time-series trend analysis.

---

The `$expMovingAvg` operator inside `$setWindowFields` calculates an exponential moving average (EMA), where more recent data points are weighted more heavily than older ones. This is distinct from a simple moving average which weights all values equally, making EMA more responsive to recent changes.

## How Exponential Moving Average Works

EMA gives exponentially decreasing weights to older observations. The formula is:

```text
EMA(t) = alpha * value(t) + (1 - alpha) * EMA(t-1)
```

Where alpha is a smoothing factor between 0 and 1. Higher alpha means more weight on recent values (more reactive), lower alpha means more weight on historical values (smoother).

## $expMovingAvg Syntax

You can specify the EMA either by a `N` value (number of historical periods) or an explicit `alpha`:

```javascript
// Using N (MongoDB computes alpha as 2 / (N + 1))
$expMovingAvg: {
  input: "<expression>",
  N: <positive integer>
}

// Using explicit alpha
$expMovingAvg: {
  input: "<expression>",
  alpha: <float between 0 and 1>
}
```

## Basic EMA with N Periods

Calculate a 7-period EMA of daily prices:

```javascript
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$symbol",
      sortBy: { date: 1 },
      output: {
        ema7: {
          $expMovingAvg: {
            input: "$close",
            N: 7
          }
        },
        ema14: {
          $expMovingAvg: {
            input: "$close",
            N: 14
          }
        }
      }
    }
  }
])
```

For N=7, MongoDB uses alpha = 2/(7+1) = 0.25. This means the most recent value gets 25% of the weight.

## Using Explicit Alpha

For full control over the smoothing factor, use `alpha` directly:

```javascript
db.metrics.aggregate([
  {
    $setWindowFields: {
      sortBy: { timestamp: 1 },
      output: {
        smoothedValue: {
          $expMovingAvg: {
            input: "$value",
            alpha: 0.3
          }
        }
      }
    }
  }
])
```

Alpha of 0.3 means the current value gets 30% weight and prior EMA gets 70% weight.

## Comparing EMA to Simple Moving Average

Run both in the same stage to compare:

```javascript
db.dailyRevenue.aggregate([
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        sma7: {
          $avg: "$revenue",
          window: { documents: [-6, 0] }
        },
        ema7: {
          $expMovingAvg: {
            input: "$revenue",
            N: 7
          }
        }
      }
    }
  }
])
```

The EMA reacts faster to recent spikes; the SMA reacts more slowly.

## EMA Within Partitions

Compute EMA per product category:

```javascript
db.productSales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { week: 1 },
      output: {
        weeklyEma: {
          $expMovingAvg: {
            input: "$unitsSold",
            N: 4
          }
        }
      }
    }
  }
])
```

Each category's EMA resets independently.

## Detecting Trend Changes

Use two EMAs to identify trend changes - when the short EMA crosses the long EMA:

```javascript
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$symbol",
      sortBy: { date: 1 },
      output: {
        ema12: { $expMovingAvg: { input: "$close", N: 12 } },
        ema26: { $expMovingAvg: { input: "$close", N: 26 } }
      }
    }
  },
  {
    $addFields: {
      macdLine: { $subtract: ["$ema12", "$ema26"] }
    }
  }
])
```

This computes the MACD line, a popular technical indicator in financial analysis.

## Summary

`$expMovingAvg` inside `$setWindowFields` brings exponential moving average calculations to MongoDB aggregation pipelines. Use the `N` parameter for a natural period-based smoothing factor, or `alpha` for precise control. EMA is preferred over simple moving averages in trend detection scenarios because it reacts more quickly to recent changes, making it particularly useful for time-series monitoring, financial data analysis, and real-time metric smoothing.
