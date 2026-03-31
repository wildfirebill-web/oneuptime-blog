# How to Use $shift for Lag and Lead Calculations in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Window Functions, Lag, Lead, Aggregation

Description: Learn how to use MongoDB's $shift window operator to access values from previous or future rows within a partition for lag and lead calculations.

---

## What Is $shift in MongoDB

`$shift` is a window operator in MongoDB's `$setWindowFields` (5.0+) that accesses a field value from a row offset from the current row within a partition. A negative offset looks back (lag), a positive offset looks forward (lead). This is equivalent to `LAG()` and `LEAD()` functions in SQL.

## Basic Syntax

```javascript
{
  $setWindowFields: {
    partitionBy: "$category",
    sortBy: { date: 1 },
    output: {
      previousValue: {
        $shift: {
          output: "$fieldName",
          by: -1,              // -1 = one row back (lag)
          default: null        // value if row doesn't exist
        }
      },
      nextValue: {
        $shift: {
          output: "$fieldName",
          by: 1,               // +1 = one row forward (lead)
          default: null
        }
      }
    }
  }
}
```

## Setup - Monthly Revenue Dataset

```javascript
db.monthlyRevenue.insertMany([
  { product: "Widget", year: 2025, month: 1,  revenue: 50000 },
  { product: "Widget", year: 2025, month: 2,  revenue: 53000 },
  { product: "Widget", year: 2025, month: 3,  revenue: 48000 },
  { product: "Widget", year: 2025, month: 4,  revenue: 62000 },
  { product: "Widget", year: 2025, month: 5,  revenue: 71000 },
  { product: "Widget", year: 2025, month: 6,  revenue: 68000 },
  { product: "Gadget", year: 2025, month: 1,  revenue: 30000 },
  { product: "Gadget", year: 2025, month: 2,  revenue: 28000 },
  { product: "Gadget", year: 2025, month: 3,  revenue: 35000 },
  { product: "Gadget", year: 2025, month: 4,  revenue: 40000 },
  { product: "Gadget", year: 2025, month: 5,  revenue: 38000 },
  { product: "Gadget", year: 2025, month: 6,  revenue: 42000 }
]);
```

## Example 1 - Month-over-Month Change (Lag)

Calculate revenue change from the previous month:

```javascript
db.monthlyRevenue.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$product",
      sortBy: { year: 1, month: 1 },
      output: {
        previousMonthRevenue: {
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
      momChange: {
        $cond: {
          if: { $ne: ["$previousMonthRevenue", null] },
          then: { $subtract: ["$revenue", "$previousMonthRevenue"] },
          else: null
        }
      },
      momChangePct: {
        $cond: {
          if: { $and: [{ $ne: ["$previousMonthRevenue", null] }, { $ne: ["$previousMonthRevenue", 0] }] },
          then: {
            $multiply: [
              { $divide: [
                { $subtract: ["$revenue", "$previousMonthRevenue"] },
                "$previousMonthRevenue"
              ]},
              100
            ]
          },
          else: null
        }
      }
    }
  },
  { $sort: { product: 1, year: 1, month: 1 } }
]);
```

## Example 2 - Look-Ahead with Lead

Show the next month's forecast alongside current data:

```javascript
db.monthlyRevenue.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$product",
      sortBy: { year: 1, month: 1 },
      output: {
        nextMonthRevenue: {
          $shift: {
            output: "$revenue",
            by: 1,
            default: null
          }
        },
        twoMonthsAheadRevenue: {
          $shift: {
            output: "$revenue",
            by: 2,
            default: null
          }
        }
      }
    }
  },
  {
    $addFields: {
      revenueProjected: {
        $cond: {
          if: { $ne: ["$nextMonthRevenue", null] },
          then: { $gt: ["$nextMonthRevenue", "$revenue"] },
          else: null
        }
      }
    }
  }
]);
```

## Example 3 - Year-over-Year Comparison Using Large Offset

For monthly data, shift by 12 to get the same month in the prior year:

```javascript
// Multi-year dataset
db.monthlyRevenue.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$product",
      sortBy: { year: 1, month: 1 },
      output: {
        sameMonthLastYear: {
          $shift: {
            output: "$revenue",
            by: -12,
            default: null
          }
        }
      }
    }
  },
  {
    $addFields: {
      yoyChange: {
        $cond: {
          if: { $ne: ["$sameMonthLastYear", null] },
          then: { $subtract: ["$revenue", "$sameMonthLastYear"] },
          else: null
        }
      }
    }
  }
]);
```

## Example 4 - Detecting Trend Reversals

Find months where revenue changed direction (from growth to decline or vice versa):

```javascript
db.monthlyRevenue.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$product",
      sortBy: { year: 1, month: 1 },
      output: {
        prevRevenue: {
          $shift: { output: "$revenue", by: -1, default: null }
        },
        nextRevenue: {
          $shift: { output: "$revenue", by: 1, default: null }
        }
      }
    }
  },
  {
    $addFields: {
      isTroughOrPeak: {
        $cond: {
          if: {
            $and: [
              { $ne: ["$prevRevenue", null] },
              { $ne: ["$nextRevenue", null] }
            ]
          },
          then: {
            $or: [
              // Peak: higher than both neighbors
              { $and: [
                { $gt: ["$revenue", "$prevRevenue"] },
                { $gt: ["$revenue", "$nextRevenue"] }
              ]},
              // Trough: lower than both neighbors
              { $and: [
                { $lt: ["$revenue", "$prevRevenue"] },
                { $lt: ["$revenue", "$nextRevenue"] }
              ]}
            ]
          },
          else: false
        }
      }
    }
  },
  { $match: { isTroughOrPeak: true } }
]);
```

## Example 5 - Session Analysis with Default Value

Use a default value to handle first-row lag gracefully in session data:

```javascript
db.userSessions.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$userId",
      sortBy: { startTime: 1 },
      output: {
        previousSessionEnd: {
          $shift: {
            output: "$endTime",
            by: -1,
            default: "$$REMOVE"  // remove field if no previous row
          }
        }
      }
    }
  },
  {
    $addFields: {
      gapFromPreviousSession: {
        $cond: {
          if: { $ifNull: ["$previousSessionEnd", false] },
          then: { $subtract: ["$startTime", "$previousSessionEnd"] },
          else: null
        }
      }
    }
  }
]);
```

## Summary

`$shift` in MongoDB's `$setWindowFields` provides lag and lead functionality, letting you compare each row against preceding or following rows in the same partition. Use negative `by` values for lag (previous row comparisons), positive values for lead (forward-looking analysis), and the `default` option to handle partition boundaries cleanly. Common use cases include month-over-month change calculations, year-over-year comparisons, trend reversal detection, and session gap analysis.
