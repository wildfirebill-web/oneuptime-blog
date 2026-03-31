# How to Use $shift to Access Adjacent Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Shift, Analytics

Description: Learn how to use the $shift window operator inside $setWindowFields to access field values from adjacent documents in a sorted partition for lag and lead calculations.

---

The `$shift` operator in MongoDB's `$setWindowFields` stage lets you look forward or backward in a sorted document sequence to access values from neighboring documents. It is equivalent to SQL's `LAG()` and `LEAD()` window functions, enabling period-over-period comparisons and sequential data analysis.

## $shift Syntax

```javascript
{
  $shift: {
    output: "<expression>",
    by: <integer>,      // positive = look ahead, negative = look behind
    default: <value>    // returned when the offset is out of bounds
  }
}
```

- `by: -1` - accesses the previous document (LAG)
- `by: 1` - accesses the next document (LEAD)
- `by: -N` - accesses N documents before the current one

## Basic Example - Previous Day Revenue

Calculate the change in daily revenue compared to the previous day:

```javascript
db.dailyRevenue.aggregate([
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        previousRevenue: {
          $shift: {
            output: "$revenue",
            by: -1,
            default: null
          }
        }
      }
    }
  },
  {
    $addFields: {
      revenueChange: {
        $cond: {
          if: { $ne: ["$previousRevenue", null] },
          then: { $subtract: ["$revenue", "$previousRevenue"] },
          else: null
        }
      }
    }
  }
])
```

Output:

```json
[
  { "date": "2024-01-01", "revenue": 1200, "previousRevenue": null, "revenueChange": null },
  { "date": "2024-01-02", "revenue": 1500, "previousRevenue": 1200, "revenueChange": 300 },
  { "date": "2024-01-03", "revenue": 1350, "previousRevenue": 1500, "revenueChange": -150 }
]
```

## Look-Ahead with Positive Offset

Use a positive `by` value to look at the next document:

```javascript
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        nextDayPrice: {
          $shift: {
            output: "$close",
            by: 1,
            default: null
          }
        }
      }
    }
  },
  {
    $addFields: {
      priceDirection: {
        $cond: {
          if: { $gt: ["$nextDayPrice", "$close"] },
          then: "up",
          else: {
            $cond: {
              if: { $lt: ["$nextDayPrice", "$close"] },
              then: "down",
              else: "flat"
            }
          }
        }
      }
    }
  }
])
```

## Shift Within Partitions

Combine `partitionBy` with `$shift` to compare within groups:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$salesRepId",
      sortBy: { month: 1 },
      output: {
        prevMonthSales: {
          $shift: {
            output: "$totalSales",
            by: -1,
            default: 0
          }
        }
      }
    }
  },
  {
    $addFields: {
      monthOverMonthGrowth: {
        $cond: {
          if: { $gt: ["$prevMonthSales", 0] },
          then: {
            $multiply: [
              { $divide: [
                { $subtract: ["$totalSales", "$prevMonthSales"] },
                "$prevMonthSales"
              ]},
              100
            ]
          },
          else: null
        }
      }
    }
  }
])
```

## Multi-Period Shift

Look back multiple periods to compare with two periods ago:

```javascript
db.metrics.aggregate([
  {
    $setWindowFields: {
      sortBy: { week: 1 },
      output: {
        twoWeeksAgo: {
          $shift: {
            output: "$activeUsers",
            by: -2,
            default: null
          }
        }
      }
    }
  }
])
```

## Using a Default Value

The `default` field controls what to return when the offset goes out of the partition boundary:

```javascript
prevValue: {
  $shift: {
    output: "$value",
    by: -1,
    default: "N/A"    // string default
  }
}
```

## Summary

`$shift` brings LAG and LEAD functionality to MongoDB aggregation. Use negative `by` values to reference previous documents and positive values to reference upcoming ones. Combined with `partitionBy`, you can calculate period-over-period changes independently within groups. The `default` parameter handles boundary conditions where no adjacent document exists, preventing null propagation errors in downstream calculations.
