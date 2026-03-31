# How to Build a Retention Analysis in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Retention, Cohort

Description: Learn how to build a user retention analysis in MongoDB by tracking how many users return in subsequent periods after their first activity.

---

## Introduction

Retention analysis measures how many users from an original cohort continue to be active in later periods. It differs from cohort analysis in that it focuses specifically on re-engagement - did a user who was active in week 1 also return in week 2, 3, and beyond?

## Data Model

Assume a `sessions` collection recording user activity:

```json
{
  "_id": ObjectId("..."),
  "userId": "user_77",
  "sessionDate": ISODate("2025-08-05T10:00:00Z")
}
```

## Step 1 - Find Each User's First Active Week

Compute the cohort week (the week of each user's first session):

```javascript
db.sessions.aggregate([
  {
    $group: {
      _id: "$userId",
      firstSession: { $min: "$sessionDate" }
    }
  },
  {
    $project: {
      userId: "$_id",
      cohortWeek: {
        $dateTrunc: { date: "$firstSession", unit: "week" }
      }
    }
  },
  { $out: "user_cohorts" }
])
```

## Step 2 - Join Sessions with Cohort Data

Join each session back to the user's cohort week and compute the week offset:

```javascript
db.sessions.aggregate([
  {
    $lookup: {
      from: "user_cohorts",
      localField: "userId",
      foreignField: "userId",
      as: "cohort"
    }
  },
  { $unwind: "$cohort" },
  {
    $project: {
      userId: 1,
      cohortWeek: "$cohort.cohortWeek",
      sessionWeek: { $dateTrunc: { date: "$sessionDate", unit: "week" } }
    }
  },
  {
    $project: {
      userId: 1,
      cohortWeek: 1,
      weekOffset: {
        $floor: {
          $divide: [
            { $subtract: ["$sessionWeek", "$cohortWeek"] },
            1000 * 60 * 60 * 24 * 7
          ]
        }
      }
    }
  }
])
```

## Step 3 - Count Retained Users per Cohort-Week Offset

```javascript
db.sessions.aggregate([
  // ... (steps 1 and 2 embedded or from $lookup)
  {
    $group: {
      _id: { cohortWeek: "$cohortWeek", weekOffset: "$weekOffset" },
      retainedUsers: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      cohortWeek: "$_id.cohortWeek",
      weekOffset: "$_id.weekOffset",
      retainedCount: { $size: "$retainedUsers" }
    }
  },
  { $sort: { cohortWeek: 1, weekOffset: 1 } }
])
```

## Calculating Retention Rate

After retrieving both cohort sizes (week offset 0) and retention counts (offsets 1, 2, ...), calculate rates in application code:

```javascript
function retentionRate(cohortSize, retainedCount) {
  if (cohortSize === 0) return 0;
  return (retainedCount / cohortSize * 100).toFixed(1);
}
```

A typical output might show: Week 1 retention 55%, Week 2 retention 38%, Week 4 retention 22%.

## Index Recommendations

```javascript
db.sessions.createIndex({ userId: 1, sessionDate: 1 })
db.user_cohorts.createIndex({ userId: 1 })
```

## Summary

Retention analysis in MongoDB involves three steps: computing each user's cohort week via `$group` and `$min`, joining session records to cohort dates via `$lookup`, and grouping by cohort week and week offset to count unique retained users. Materializing the `user_cohorts` collection avoids re-computing first sessions on every query. Indexes on `userId` and `sessionDate` are essential for join and grouping performance.
