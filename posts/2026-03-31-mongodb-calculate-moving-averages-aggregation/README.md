# How to Calculate Moving Averages in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Moving Average, Analytics

Description: Learn how to calculate moving averages in MongoDB using $setWindowFields and sliding window expressions for time-series and trend analysis.

---

Moving averages smooth out short-term fluctuations in data to reveal underlying trends. MongoDB's `$setWindowFields` stage, introduced in MongoDB 5.0, provides a native way to compute moving averages without complex workarounds.

## Understanding the $setWindowFields Stage

`$setWindowFields` lets you apply window functions over a range of documents sorted by a field - similar to SQL's `OVER (ORDER BY ... ROWS BETWEEN ...)` syntax. It does not collapse documents, so you retain all original fields while adding computed values.

## Basic 3-Day Moving Average

Suppose you have a `sales` collection with daily revenue records:

```javascript
db.sales.insertMany([
  { date: ISODate("2024-01-01"), revenue: 1200 },
  { date: ISODate("2024-01-02"), revenue: 980 },
  { date: ISODate("2024-01-03"), revenue: 1450 },
  { date: ISODate("2024-01-04"), revenue: 1100 },
  { date: ISODate("2024-01-05"), revenue: 1600 }
])
```

Calculate a 3-day trailing moving average:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        movingAvg3Day: {
          $avg: "$revenue",
          window: {
            documents: [-2, 0]
          }
        }
      }
    }
  },
  {
    $project: {
      date: 1,
      revenue: 1,
      movingAvg3Day: { $round: ["$movingAvg3Day", 2] }
    }
  }
])
```

The `documents: [-2, 0]` window means: the current document and the 2 preceding documents.

## Range-Based Moving Average

Instead of a fixed document count, use a value-based range. For a 7-day rolling average where dates might have gaps:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        rolling7DayAvg: {
          $avg: "$revenue",
          window: {
            range: [-6, 0],
            unit: "day"
          }
        }
      }
    }
  }
])
```

With `range` and `unit`, MongoDB uses actual date values rather than document position, correctly handling missing days.

## Partitioned Moving Average

When data has multiple categories (e.g., revenue by product), use `partitionBy` to compute moving averages per group:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$productId",
      sortBy: { date: 1 },
      output: {
        avgRevenue: {
          $avg: "$revenue",
          window: { documents: [-4, 0] }
        }
      }
    }
  }
])
```

Each partition maintains its own window independently, so product A's moving average does not bleed into product B's calculation.

## Combining with $match and $project

For production use, filter data before computing and project only the needed fields:

```javascript
db.sales.aggregate([
  { $match: { date: { $gte: ISODate("2024-01-01") } } },
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        ma7: { $avg: "$revenue", window: { range: [-6, 0], unit: "day" } },
        ma30: { $avg: "$revenue", window: { range: [-29, 0], unit: "day" } }
      }
    }
  },
  {
    $project: {
      _id: 0,
      date: 1,
      revenue: 1,
      ma7: { $round: ["$ma7", 2] },
      ma30: { $round: ["$ma30", 2] }
    }
  }
])
```

## Performance Considerations

- Add an index on the sort field (`date` in these examples) to avoid a full collection scan.
- Use `$match` as the first stage to limit the dataset before windowing.
- `$setWindowFields` requires sorting, which can be memory-intensive on large datasets - ensure `allowDiskUse: true` if needed.
- For extremely large time-series collections, consider using MongoDB time-series collections, which have optimized bucket storage for window queries.

## Summary

MongoDB's `$setWindowFields` stage with `$avg` and a sliding `documents` or `range` window makes moving averages straightforward. Use `documents` for position-based windows and `range` with `unit` for time-based rolling windows that handle gaps correctly. Combine with `partitionBy` to compute per-group moving averages in a single pipeline pass.
