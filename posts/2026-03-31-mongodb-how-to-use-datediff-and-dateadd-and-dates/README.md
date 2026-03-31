# How to Use $dateDiff, $dateAdd, and $dateSubtract in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Date Arithmetic, Database

Description: Learn how to use $dateDiff, $dateAdd, and $dateSubtract in MongoDB aggregation pipelines to calculate date differences and perform arithmetic on date fields.

---

## Overview

MongoDB 5.0 introduced `$dateDiff`, `$dateAdd`, and `$dateSubtract` operators for performing date arithmetic directly in aggregation pipelines. These operators eliminate the need to pull dates into application code for time-based calculations like subscription durations, deadlines, or scheduling windows.

## Using $dateDiff to Calculate Date Differences

The `$dateDiff` operator calculates the difference between two dates in the specified time unit.

```javascript
db.subscriptions.aggregate([
  {
    $project: {
      userId: 1,
      startDate: 1,
      endDate: 1,
      durationDays: {
        $dateDiff: {
          startDate: "$startDate",
          endDate: "$endDate",
          unit: "day"
        }
      }
    }
  }
])
```

Supported units: `year`, `quarter`, `month`, `week`, `day`, `hour`, `minute`, `second`, `millisecond`.

Calculate age in years:

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      ageYears: {
        $dateDiff: {
          startDate: "$birthDate",
          endDate: "$$NOW",
          unit: "year"
        }
      }
    }
  }
])
```

## Using $dateAdd to Add Time to a Date

The `$dateAdd` operator adds a specified amount of time to a date:

```javascript
db.tasks.aggregate([
  {
    $project: {
      taskName: 1,
      createdAt: 1,
      dueDate: {
        $dateAdd: {
          startDate: "$createdAt",
          unit: "day",
          amount: 30
        }
      }
    }
  }
])
```

You can also use a field reference as the `amount`:

```javascript
db.trials.aggregate([
  {
    $project: {
      userId: 1,
      startDate: 1,
      expiryDate: {
        $dateAdd: {
          startDate: "$startDate",
          unit: "day",
          amount: "$trialDays"
        }
      }
    }
  }
])
```

## Using $dateSubtract to Subtract Time from a Date

The `$dateSubtract` operator works identically to `$dateAdd` but subtracts time instead:

```javascript
db.reports.aggregate([
  {
    $project: {
      reportDate: 1,
      lookbackStart: {
        $dateSubtract: {
          startDate: "$reportDate",
          unit: "month",
          amount: 3
        }
      }
    }
  }
])
```

Find documents where a date was within the last 7 days:

```javascript
db.events.aggregate([
  {
    $match: {
      $expr: {
        $gte: [
          "$eventDate",
          {
            $dateSubtract: {
              startDate: "$$NOW",
              unit: "day",
              amount: 7
            }
          }
        ]
      }
    }
  }
])
```

## Timezone Support

All three operators accept an optional `timezone` parameter for daylight saving time awareness:

```javascript
db.appointments.aggregate([
  {
    $project: {
      appointmentDate: 1,
      reminderDate: {
        $dateSubtract: {
          startDate: "$appointmentDate",
          unit: "hour",
          amount: 24,
          timezone: "America/New_York"
        }
      }
    }
  }
])
```

## Combining Date Arithmetic in a Pipeline

Calculate whether a subscription is active, expired, or expiring soon:

```javascript
db.subscriptions.aggregate([
  {
    $addFields: {
      expiryDate: {
        $dateAdd: {
          startDate: "$startDate",
          unit: "month",
          amount: "$durationMonths"
        }
      }
    }
  },
  {
    $addFields: {
      daysUntilExpiry: {
        $dateDiff: {
          startDate: "$$NOW",
          endDate: "$expiryDate",
          unit: "day"
        }
      }
    }
  },
  {
    $addFields: {
      status: {
        $switch: {
          branches: [
            { case: { $lt: ["$daysUntilExpiry", 0] }, then: "expired" },
            { case: { $lt: ["$daysUntilExpiry", 7] }, then: "expiring_soon" }
          ],
          default: "active"
        }
      }
    }
  }
])
```

## Summary

MongoDB's `$dateDiff`, `$dateAdd`, and `$dateSubtract` operators provide a clean, pipeline-native way to perform date arithmetic. They support all major time units and timezone-aware calculations, making them ideal for subscription management, scheduling, SLA tracking, and time window filtering. Combined with `$$NOW` and conditional expressions, they enable rich, self-contained temporal logic in aggregation pipelines.
