# How to Calculate Churn Rate in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Churn, Metric

Description: Learn how to calculate monthly churn rate in MongoDB by comparing active subscribers at the start and end of each period using aggregation.

---

## Introduction

Churn rate measures the percentage of customers who stopped using your service in a given period. For subscription businesses, monthly churn is typically defined as: (customers who churned during the month) / (customers active at the start of the month). MongoDB's aggregation pipeline can compute this by analyzing subscription status changes over time.

## Sample Data Structure

Assume a `subscriptions` collection:

```json
{
  "_id": ObjectId("..."),
  "customerId": "cust_202",
  "status": "cancelled",
  "startDate": ISODate("2024-11-01T00:00:00Z"),
  "cancelDate": ISODate("2025-03-15T00:00:00Z"),
  "plan": "pro"
}
```

## Counting Active Subscribers at Period Start

Count customers with an active subscription at the beginning of a month:

```javascript
const periodStart = ISODate("2025-03-01T00:00:00Z");
const periodEnd = ISODate("2025-04-01T00:00:00Z");

db.subscriptions.aggregate([
  {
    $match: {
      startDate: { $lt: periodStart },
      $or: [
        { cancelDate: { $exists: false } },
        { cancelDate: { $gte: periodStart } }
      ]
    }
  },
  {
    $count: "activeAtStart"
  }
])
```

## Counting Churned Customers in the Period

Count customers who cancelled within the month:

```javascript
db.subscriptions.aggregate([
  {
    $match: {
      cancelDate: {
        $gte: periodStart,
        $lt: periodEnd
      }
    }
  },
  {
    $count: "churned"
  }
])
```

## Computing Churn Rate per Month Using a Single Pipeline

Combine both computations with `$facet`:

```javascript
db.subscriptions.aggregate([
  {
    $facet: {
      activeAtStart: [
        {
          $match: {
            startDate: { $lt: ISODate("2025-03-01T00:00:00Z") },
            $or: [
              { cancelDate: { $exists: false } },
              { cancelDate: { $gte: ISODate("2025-03-01T00:00:00Z") } }
            ]
          }
        },
        { $count: "count" }
      ],
      churned: [
        {
          $match: {
            cancelDate: {
              $gte: ISODate("2025-03-01T00:00:00Z"),
              $lt: ISODate("2025-04-01T00:00:00Z")
            }
          }
        },
        { $count: "count" }
      ]
    }
  },
  {
    $project: {
      activeAtStart: { $arrayElemAt: ["$activeAtStart.count", 0] },
      churned: { $arrayElemAt: ["$churned.count", 0] }
    }
  },
  {
    $project: {
      activeAtStart: 1,
      churned: 1,
      churnRate: {
        $multiply: [
          { $divide: ["$churned", "$activeAtStart"] },
          100
        ]
      }
    }
  }
])
```

## Computing Churn Across Multiple Months

To see monthly churn trends, group cancelled subscriptions by cancel month and calculate the rate using pre-computed monthly active counts:

```javascript
db.subscriptions.aggregate([
  {
    $match: { cancelDate: { $exists: true } }
  },
  {
    $group: {
      _id: {
        year: { $year: "$cancelDate" },
        month: { $month: "$cancelDate" }
      },
      churned: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

Join the churned counts with pre-stored monthly active counts to calculate the churn rate for each month.

## Index Recommendation

```javascript
db.subscriptions.createIndex({ startDate: 1, cancelDate: 1 })
```

## Summary

Calculating churn rate in MongoDB uses `$match` to identify customers active at the start of a period and those who cancelled within it, then divides churned count by active-at-start count. The `$facet` stage allows computing both numbers in a single aggregation. For multi-month trends, group by cancel month and combine with pre-aggregated active subscriber counts. Indexing on `startDate` and `cancelDate` is essential for performance.
