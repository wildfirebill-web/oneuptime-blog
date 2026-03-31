# How to Calculate the Difference Between Two Dates in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Date, Aggregation, Arithmetic, Duration

Description: Learn how to calculate the difference between two dates in MongoDB using $dateDiff, $subtract, and custom expressions for duration calculations.

---

## Overview

Calculating time differences is essential for measuring durations such as session length, response times, or days since last purchase. MongoDB 5.0 introduced the `$dateDiff` operator for readable, unit-aware date subtraction.

## Using $dateDiff (MongoDB 5.0+)

`$dateDiff` returns the integer difference between two dates in the specified unit.

```javascript
db.sessions.aggregate([
  {
    $project: {
      durationMinutes: {
        $dateDiff: {
          startDate: "$startedAt",
          endDate: "$endedAt",
          unit: "minute"
        }
      }
    }
  }
]);
```

## Computing Age in Years

```javascript
db.users.aggregate([
  {
    $project: {
      ageYears: {
        $dateDiff: {
          startDate: "$dateOfBirth",
          endDate: "$$NOW",
          unit: "year"
        }
      }
    }
  }
]);
```

## Days Since Last Activity

```javascript
db.users.aggregate([
  {
    $project: {
      daysSinceLogin: {
        $dateDiff: {
          startDate: "$lastLoginAt",
          endDate: "$$NOW",
          unit: "day"
        }
      }
    }
  }
]);
```

## Using $subtract for Millisecond Difference (Pre-5.0)

Before MongoDB 5.0, subtract two date fields to get a millisecond value, then convert.

```javascript
db.sessions.aggregate([
  {
    $project: {
      durationMs: { $subtract: ["$endedAt", "$startedAt"] },
      durationSeconds: {
        $divide: [{ $subtract: ["$endedAt", "$startedAt"] }, 1000]
      },
      durationMinutes: {
        $divide: [{ $subtract: ["$endedAt", "$startedAt"] }, 60000]
      }
    }
  }
]);
```

## Filtering by Duration

Find sessions that lasted more than 30 minutes.

```javascript
db.sessions.aggregate([
  {
    $match: {
      $expr: {
        $gt: [
          {
            $dateDiff: {
              startDate: "$startedAt",
              endDate: "$endedAt",
              unit: "minute"
            }
          },
          30
        ]
      }
    }
  }
]);
```

## Timezone-Aware Day Boundaries

When measuring in days, use a timezone to align with calendar day boundaries.

```javascript
db.events.aggregate([
  {
    $project: {
      daysElapsed: {
        $dateDiff: {
          startDate: "$eventDate",
          endDate: "$$NOW",
          unit: "day",
          timezone: "America/New_York"
        }
      }
    }
  }
]);
```

## Summary

MongoDB 5.0's `$dateDiff` offers clean, unit-aware date subtraction. For older versions, use `$subtract` to get a millisecond delta and then divide to convert units. Combine with `$match` and `$expr` to filter documents by computed durations.
