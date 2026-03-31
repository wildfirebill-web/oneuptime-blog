# How to Calculate Percentages and Ratios in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Percentage, Ratio

Description: Learn how to calculate percentages and ratios in MongoDB aggregation pipelines using $divide, $multiply, and $facet for total-relative metrics.

---

## Introduction

Calculating percentages and ratios in MongoDB requires combining per-group values with a total. Since MongoDB's aggregation pipeline processes documents in stages, you need to either use `$facet` to compute totals alongside group sums, or perform two passes over the data.

## Sample Data

Assume a `surveys` collection with user responses:

```json
{
  "_id": ObjectId("..."),
  "userId": "user_301",
  "rating": 5,
  "category": "support",
  "submittedAt": ISODate("2025-07-10T09:00:00Z")
}
```

## Percentage of Total Using $facet

Use `$facet` to compute both the grand total and per-group counts in one pipeline pass:

```javascript
db.surveys.aggregate([
  {
    $facet: {
      byRating: [
        { $group: { _id: "$rating", count: { $sum: 1 } } },
        { $sort: { _id: 1 } }
      ],
      total: [
        { $count: "n" }
      ]
    }
  },
  { $unwind: "$total" },
  {
    $project: {
      ratings: {
        $map: {
          input: "$byRating",
          as: "r",
          in: {
            rating: "$$r._id",
            count: "$$r.count",
            percentage: {
              $multiply: [
                { $divide: ["$$r.count", "$total.n"] },
                100
              ]
            }
          }
        }
      },
      total: "$total.n"
    }
  }
])
```

## Ratio Between Two Metrics

Calculate the ratio of positive ratings (4-5) to negative ratings (1-2):

```javascript
db.surveys.aggregate([
  {
    $facet: {
      positive: [
        { $match: { rating: { $gte: 4 } } },
        { $count: "n" }
      ],
      negative: [
        { $match: { rating: { $lte: 2 } } },
        { $count: "n" }
      ]
    }
  },
  {
    $project: {
      positiveCount: { $arrayElemAt: ["$positive.n", 0] },
      negativeCount: { $arrayElemAt: ["$negative.n", 0] }
    }
  },
  {
    $project: {
      positiveCount: 1,
      negativeCount: 1,
      ratio: {
        $cond: {
          if: { $gt: ["$negativeCount", 0] },
          then: { $divide: ["$positiveCount", "$negativeCount"] },
          else: null
        }
      }
    }
  }
])
```

Using `$cond` prevents division-by-zero errors when the denominator might be zero.

## Percentage Share per Group

Calculate each category's share of total survey responses:

```javascript
db.surveys.aggregate([
  {
    $group: {
      _id: "$category",
      count: { $sum: 1 }
    }
  },
  {
    $group: {
      _id: null,
      categories: { $push: { category: "$_id", count: "$count" } },
      total: { $sum: "$count" }
    }
  },
  {
    $project: {
      breakdown: {
        $map: {
          input: "$categories",
          as: "cat",
          in: {
            category: "$$cat.category",
            count: "$$cat.count",
            share: {
              $round: [
                { $multiply: [{ $divide: ["$$cat.count", "$total"] }, 100] },
                2
              ]
            }
          }
        }
      }
    }
  }
])
```

`$round` trims the percentage to two decimal places for clean output.

## Summary

Calculating percentages and ratios in MongoDB aggregation relies on computing the denominator alongside the numerator. Use `$facet` when you need the total and per-group values in the same query, and `$map` with `$divide` and `$multiply` to compute the final percentage for each group. Always guard division with `$cond` to handle zero-denominator cases. The double-`$group` pattern also works when you want to accumulate group totals and the grand total in sequence.
