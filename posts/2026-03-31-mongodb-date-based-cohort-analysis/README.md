# How to Build a Date-Based Cohort Analysis in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Cohort, Analytics, Aggregation, Retention

Description: Build date-based cohort analysis in MongoDB to measure user retention, churn, and engagement by grouping users by their signup or first-event date.

---

## Overview

Cohort analysis groups users by a shared characteristic - typically their signup date - and tracks their behavior over subsequent periods. In MongoDB, you can implement cohort analysis using `$group` to define cohorts, `$lookup` or `$setWindowFields` to find subsequent events, and aggregation to compute retention rates for each cohort.

## Data Model

Two collections: `users` for cohort assignment and `events` for activity tracking.

```javascript
// users
{ _id: "user_001", createdAt: new Date("2026-01-15"), plan: "free" }

// events
{ _id: ObjectId(), userId: "user_001", eventType: "login", occurredAt: new Date("2026-02-10") }
```

## Step 1: Assign Users to Monthly Cohorts

Group users by the month they signed up.

```javascript
const cohortAssignments = await db.collection("users").aggregate([
  {
    $project: {
      cohortMonth: {
        $dateToString: { format: "%Y-%m", date: "$createdAt" },
      },
      createdAt: 1,
    },
  },
  {
    $group: {
      _id: "$cohortMonth",
      users: { $push: "$_id" },
      cohortSize: { $sum: 1 },
    },
  },
  { $sort: { _id: 1 } },
]).toArray();
```

## Step 2: Compute Retention by Cohort

Find which users were active in each subsequent month after their cohort month.

```javascript
const retentionData = await db.collection("users").aggregate([
  {
    $addFields: {
      cohortMonth: { $dateToString: { format: "%Y-%m", date: "$createdAt" } },
    },
  },
  {
    $lookup: {
      from: "events",
      let: { userId: "$_id", cohort: { $dateToString: { format: "%Y-%m", date: "$createdAt" } } },
      pipeline: [
        {
          $match: {
            $expr: { $eq: ["$userId", "$$userId"] },
            eventType: "login",
          },
        },
        {
          $project: {
            eventMonth: { $dateToString: { format: "%Y-%m", date: "$occurredAt" } },
          },
        },
        { $group: { _id: "$eventMonth" } },
      ],
      as: "activeMonths",
    },
  },
  { $unwind: "$activeMonths" },
  {
    $project: {
      cohortMonth: 1,
      activeMonth: "$activeMonths._id",
      monthOffset: {
        $dateDiff: {
          startDate: { $dateFromString: { dateString: { $dateToString: { format: "%Y-%m", date: "$createdAt" } }, format: "%Y-%m" } },
          endDate: { $dateFromString: { dateString: "$activeMonths._id", format: "%Y-%m" } },
          unit: "month",
        },
      },
    },
  },
  {
    $group: {
      _id: { cohortMonth: "$cohortMonth", monthOffset: "$monthOffset" },
      activeUsers: { $sum: 1 },
    },
  },
  { $sort: { "_id.cohortMonth": 1, "_id.monthOffset": 1 } },
]).toArray();
```

## Step 3: Calculate Retention Rates

Merge cohort sizes with activity data to compute percentages.

```javascript
function buildRetentionMatrix(cohorts, retentionRows) {
  const cohortMap = Object.fromEntries(cohorts.map((c) => [c._id, c.cohortSize]));
  const matrix = {};

  for (const row of retentionRows) {
    const { cohortMonth, monthOffset } = row._id;
    if (!matrix[cohortMonth]) matrix[cohortMonth] = {};
    const size = cohortMap[cohortMonth] || 1;
    matrix[cohortMonth][monthOffset] = {
      activeUsers: row.activeUsers,
      retentionRate: ((row.activeUsers / size) * 100).toFixed(1) + "%",
    };
  }
  return matrix;
}
```

## Simplified Cohort Query for Month 0 vs Month 1

```javascript
const simpleCohort = await db.collection("users").aggregate([
  { $match: { createdAt: { $gte: new Date("2026-01-01") } } },
  {
    $lookup: {
      from: "events",
      localField: "_id",
      foreignField: "userId",
      as: "events",
    },
  },
  {
    $project: {
      cohortMonth: { $dateToString: { format: "%Y-%m", date: "$createdAt" } },
      returnedMonth1: {
        $gt: [
          {
            $size: {
              $filter: {
                input: "$events",
                as: "e",
                cond: {
                  $and: [
                    { $gte: ["$$e.occurredAt", new Date("2026-02-01")] },
                    { $lt: ["$$e.occurredAt", new Date("2026-03-01")] },
                  ],
                },
              },
            },
          },
          0,
        ],
      },
    },
  },
  {
    $group: {
      _id: "$cohortMonth",
      total: { $sum: 1 },
      returnedCount: { $sum: { $cond: ["$returnedMonth1", 1, 0] } },
    },
  },
]).toArray();
```

## Summary

MongoDB cohort analysis uses `$dateToString` to bucket users into cohorts, `$lookup` to join their subsequent activity events, `$dateDiff` to compute the month offset between cohort and activity, and aggregation to count active users per offset. This gives you a full retention matrix without any external analytics tools.
