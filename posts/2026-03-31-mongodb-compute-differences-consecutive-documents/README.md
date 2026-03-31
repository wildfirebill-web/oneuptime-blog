# How to Compute Differences Between Consecutive Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Analytics, Pipeline

Description: Learn how to compute differences between consecutive documents in MongoDB using $setWindowFields with $shift and subtraction to calculate deltas and changes.

---

Computing the difference between a row and the previous row is common in financial analysis, sensor monitoring, and trend detection. MongoDB 5.0's `$setWindowFields` and `$shift` operator make this straightforward.

## Using $shift to Access the Previous Document

The `$shift` operator returns a field value from a document at a relative position in the window:

```javascript
db.stockPrices.insertMany([
  { date: ISODate("2024-01-01"), ticker: "AAPL", close: 185.20 },
  { date: ISODate("2024-01-02"), ticker: "AAPL", close: 187.50 },
  { date: ISODate("2024-01-03"), ticker: "AAPL", close: 183.10 },
  { date: ISODate("2024-01-04"), ticker: "AAPL", close: 188.90 }
])
```

Calculate the day-over-day price change:

```javascript
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { date: 1 },
      output: {
        prevClose: {
          $shift: {
            output: "$close",
            by: -1,
            default: null
          }
        }
      }
    }
  },
  {
    $addFields: {
      dailyChange: {
        $cond: [
          { $ne: ["$prevClose", null] },
          { $subtract: ["$close", "$prevClose"] },
          null
        ]
      }
    }
  },
  {
    $project: {
      _id: 0,
      date: 1,
      ticker: 1,
      close: 1,
      dailyChange: { $round: ["$dailyChange", 2] }
    }
  }
])
```

`by: -1` shifts backward by one position (the previous document). Use `by: 1` to look at the next document.

## Percentage Change Between Consecutive Documents

Extend the above to compute percentage change:

```javascript
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { date: 1 },
      output: {
        prevClose: { $shift: { output: "$close", by: -1, default: null } }
      }
    }
  },
  {
    $addFields: {
      pctChange: {
        $cond: [
          { $and: [{ $ne: ["$prevClose", null] }, { $ne: ["$prevClose", 0] }] },
          {
            $multiply: [
              { $divide: [{ $subtract: ["$close", "$prevClose"] }, "$prevClose"] },
              100
            ]
          },
          null
        ]
      }
    }
  },
  { $project: { date: 1, ticker: 1, close: 1, pctChange: { $round: ["$pctChange", 2] } } }
])
```

## Differences Between Non-Adjacent Documents

To compute the change versus two periods ago, set `by: -2`:

```javascript
db.sensorReadings.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$sensorId",
      sortBy: { timestamp: 1 },
      output: {
        reading2Back: { $shift: { output: "$value", by: -2, default: null } }
      }
    }
  },
  {
    $addFields: {
      twoStepDelta: {
        $cond: [
          { $ne: ["$reading2Back", null] },
          { $subtract: ["$value", "$reading2Back"] },
          null
        ]
      }
    }
  }
])
```

## Running Cumulative Sum (Related Technique)

A related consecutive-document operation is computing a running total using `$sum` with an unbounded lower window:

```javascript
db.transactions.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$accountId",
      sortBy: { date: 1 },
      output: {
        runningBalance: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

## Performance Tips

- Index the `sortBy` field (e.g., `date`, `timestamp`) and the `partitionBy` field together as a compound index for best performance.
- Use `$match` before `$setWindowFields` to narrow the dataset.
- `$shift` is lightweight - it reads a single field from an adjacent document rather than scanning the entire window.
- For very high-cardinality partitions, consider pre-sorting with an indexed field to avoid in-memory sorts.

## Summary

MongoDB's `$shift` operator inside `$setWindowFields` provides a clean way to access previous or next document values. Combine it with `$subtract` in a subsequent `$addFields` stage to compute consecutive differences, deltas, or percentage changes across ordered time-series or sequential data.
