# How to Build a Cohort Analysis in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Cohort, Retention

Description: Learn how to build a cohort analysis in MongoDB by grouping users by signup month and tracking their activity over subsequent months.

---

## Introduction

Cohort analysis groups users by a shared characteristic - typically the month they signed up - and tracks their behavior over time. It is widely used to measure retention and understand how different user cohorts engage with a product. MongoDB's aggregation pipeline makes this possible by joining user signup dates with activity event data.

## Data Model

Two collections are needed. A `users` collection:

```json
{
  "_id": "user_55",
  "signupDate": ISODate("2025-03-10T08:00:00Z")
}
```

And an `events` collection:

```json
{
  "_id": ObjectId("..."),
  "userId": "user_55",
  "timestamp": ISODate("2025-05-22T14:00:00Z")
}
```

## Step 1 - Add Cohort Month to Events

Use `$lookup` to join events with user signup dates, then compute the cohort month and the number of months elapsed:

```javascript
db.events.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },
  {
    $project: {
      userId: 1,
      cohortMonth: {
        $dateToString: { format: "%Y-%m", date: "$user.signupDate" }
      },
      eventMonth: {
        $dateToString: { format: "%Y-%m", date: "$timestamp" }
      },
      monthsElapsed: {
        $floor: {
          $divide: [
            { $subtract: ["$timestamp", "$user.signupDate"] },
            1000 * 60 * 60 * 24 * 30
          ]
        }
      }
    }
  }
])
```

## Step 2 - Count Retained Users per Cohort-Month Pair

Group by cohort month and elapsed months, counting unique active users:

```javascript
db.events.aggregate([
  // ... (lookup and project from Step 1)
  {
    $group: {
      _id: { cohortMonth: "$cohortMonth", monthsElapsed: "$monthsElapsed" },
      activeUsers: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      cohortMonth: "$_id.cohortMonth",
      monthsElapsed: "$_id.monthsElapsed",
      activeCount: { $size: "$activeUsers" }
    }
  },
  { $sort: { cohortMonth: 1, monthsElapsed: 1 } }
])
```

The result is a table where each row represents a cohort-month bucket, making it easy to compute retention rates in application code.

## Step 3 - Compute Cohort Sizes

Get the initial size of each cohort (month 0) to calculate retention percentages:

```javascript
db.users.aggregate([
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m", date: "$signupDate" } },
      cohortSize: { $sum: 1 }
    }
  }
])
```

Divide `activeCount` at each `monthsElapsed` by the corresponding `cohortSize` to get retention rates such as 60% retained at month 1, 35% at month 3.

## Index Recommendations

```javascript
db.events.createIndex({ userId: 1, timestamp: 1 })
db.users.createIndex({ signupDate: 1 })
```

## Summary

Building a cohort analysis in MongoDB involves joining events with user signup data via `$lookup`, computing the cohort month and elapsed months via `$project`, and grouping with `$addToSet` to count unique retained users. Cross-referencing with cohort sizes enables retention rate computation. Indexes on `userId`, `timestamp`, and `signupDate` keep the pipeline performant on large datasets.
