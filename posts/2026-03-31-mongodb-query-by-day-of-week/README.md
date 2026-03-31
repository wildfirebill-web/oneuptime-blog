# How to Query by Day of Week in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Date, Aggregation, Day Of Week

Description: Learn how to query MongoDB documents by day of week using the $dayOfWeek aggregation operator and $expr to filter or group data by weekday or weekend.

---

Querying MongoDB by day of week is useful for weekly reporting, scheduling logic, and detecting patterns in time series data. MongoDB provides `$dayOfWeek` in the aggregation framework to extract the day of the week from a date field.

## How $dayOfWeek Works

`$dayOfWeek` returns a number from 1 (Sunday) to 7 (Saturday):

| Value | Day |
|-------|-----|
| 1 | Sunday |
| 2 | Monday |
| 3 | Tuesday |
| 4 | Wednesday |
| 5 | Thursday |
| 6 | Friday |
| 7 | Saturday |

## Filter Documents by Day of Week

Use `$expr` and `$dayOfWeek` in a `find` query:

```javascript
// Find all orders placed on Monday
db.orders.find({
  $expr: {
    $eq: [{ $dayOfWeek: "$createdAt" }, 2]
  }
})
```

Find orders placed on weekdays (Monday through Friday):

```javascript
db.orders.find({
  $expr: {
    $and: [
      { $gte: [{ $dayOfWeek: "$createdAt" }, 2] },
      { $lte: [{ $dayOfWeek: "$createdAt" }, 6] }
    ]
  }
})
```

Find weekend orders:

```javascript
db.orders.find({
  $expr: {
    $in: [{ $dayOfWeek: "$createdAt" }, [1, 7]]
  }
})
```

## Group by Day of Week in Aggregation

Count orders per day of the week:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: { $dayOfWeek: "$createdAt" },
      orderCount: { $sum: 1 },
      totalRevenue: { $sum: "$amount" }
    }
  },
  { $sort: { "_id": 1 } },
  {
    $addFields: {
      dayName: {
        $switch: {
          branches: [
            { case: { $eq: ["$_id", 1] }, then: "Sunday" },
            { case: { $eq: ["$_id", 2] }, then: "Monday" },
            { case: { $eq: ["$_id", 3] }, then: "Tuesday" },
            { case: { $eq: ["$_id", 4] }, then: "Wednesday" },
            { case: { $eq: ["$_id", 5] }, then: "Thursday" },
            { case: { $eq: ["$_id", 6] }, then: "Friday" },
            { case: { $eq: ["$_id", 7] }, then: "Saturday" }
          ]
        }
      }
    }
  }
])
```

## Account for Timezones

`$dayOfWeek` evaluates dates in UTC by default. For timezone-aware calculations, use the `timezone` option (MongoDB 3.6+):

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { $dayOfWeek: { date: "$createdAt", timezone: "America/New_York" } },
      count: { $sum: 1 }
    }
  }
])
```

## Python Example

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["salesdb"]

pipeline = [
    { "$match": { "status": "completed" } },
    {
        "$group": {
            "_id": { "$dayOfWeek": "$createdAt" },
            "count": { "$sum": 1 },
            "revenue": { "$sum": "$amount" }
        }
    },
    { "$sort": { "_id": 1 } }
]

results = list(db.orders.aggregate(pipeline))
days = ["", "Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"]

for row in results:
    print(f"{days[row['_id']]}: {row['count']} orders, ${row['revenue']:.2f}")
```

## Performance Notes

Queries using `$expr` with date extraction operators cannot use standard indexes efficiently. For high-volume collections, consider adding a pre-computed `dayOfWeek` field when documents are inserted:

```javascript
db.orders.insertOne({
  amount: 99.99,
  status: "completed",
  createdAt: new Date(),
  dayOfWeek: new Date().getDay() + 1  // 1=Sunday, consistent with MongoDB
})

db.orders.createIndex({ dayOfWeek: 1 })
```

## Summary

MongoDB's `$dayOfWeek` operator (returning 1 for Sunday through 7 for Saturday) enables filtering and grouping documents by day of the week. Use `$expr` in `find` queries for direct filtering, and `$group` with `$dayOfWeek` in aggregation pipelines for day-of-week analytics. For timezone-aware results, pass the `timezone` option to `$dayOfWeek`. Pre-compute the day of week field for high-volume collections to enable index-based queries.
