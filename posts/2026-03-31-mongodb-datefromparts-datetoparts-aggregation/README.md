# How to Use $dateFromParts and $dateToParts in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Date, Pipeline, Expression

Description: Learn how to construct dates from individual components with $dateFromParts and decompose dates into their parts with $dateToParts in MongoDB aggregation.

---

## Overview

MongoDB's `$dateFromParts` and `$dateToParts` operators let you work with date components directly in aggregation pipelines. `$dateFromParts` constructs a BSON Date from year, month, day and time components, while `$dateToParts` breaks a Date into its named components.

## $dateFromParts

`$dateFromParts` builds a Date value from explicit year, month, day, hour, minute, second, and millisecond fields. All fields except `year` are optional and default to their minimum values.

```javascript
db.events.aggregate([
  {
    $project: {
      constructedDate: {
        $dateFromParts: {
          year: 2025,
          month: 6,
          day: 15,
          hour: 14,
          minute: 30,
          second: 0,
          timezone: "America/New_York"
        }
      }
    }
  }
])
```

You can also use document fields as component values:

```javascript
db.appointments.aggregate([
  {
    $project: {
      scheduledAt: {
        $dateFromParts: {
          year: "$year",
          month: "$month",
          day: "$day",
          hour: "$hour",
          minute: "$minute"
        }
      }
    }
  }
])
```

## ISO Week-Based Date Construction

For ISO calendar dates, use `isoWeekYear`, `isoWeek`, and `isoDayOfWeek`:

```javascript
db.sprints.aggregate([
  {
    $project: {
      sprintStart: {
        $dateFromParts: {
          isoWeekYear: "$year",
          isoWeek: "$weekNumber",
          isoDayOfWeek: 1
        }
      }
    }
  }
])
```

## $dateToParts

`$dateToParts` decomposes a BSON Date into a document containing its individual components:

```javascript
db.orders.aggregate([
  {
    $project: {
      createdAt: 1,
      dateParts: {
        $dateToParts: {
          date: "$createdAt",
          timezone: "UTC"
        }
      }
    }
  }
])
```

This produces output like:

```javascript
{
  "dateParts": {
    "year": 2025,
    "month": 6,
    "day": 15,
    "hour": 9,
    "minute": 30,
    "second": 0,
    "millisecond": 0
  }
}
```

## Rounding Dates by Reconstructing Them

A common pattern is decomposing a date, modifying certain parts, and reconstructing it:

```javascript
db.events.aggregate([
  {
    $project: {
      timestamp: 1,
      parts: { $dateToParts: { date: "$timestamp" } }
    }
  },
  {
    $project: {
      hourStart: {
        $dateFromParts: {
          year: "$parts.year",
          month: "$parts.month",
          day: "$parts.day",
          hour: "$parts.hour"
        }
      }
    }
  }
])
```

This "rounds" each timestamp down to the start of the hour.

## Using ISO Parts

For ISO week-based decomposition, pass `iso8601: true`:

```javascript
db.tasks.aggregate([
  {
    $project: {
      dueDate: 1,
      isoParts: {
        $dateToParts: {
          date: "$dueDate",
          iso8601: true
        }
      }
    }
  }
])
```

The result includes `isoWeekYear`, `isoWeek`, and `isoDayOfWeek` instead of `year`, `month`, `day`.

## Summary

`$dateFromParts` and `$dateToParts` are complementary operators for working with date components in MongoDB aggregation. Use `$dateFromParts` to construct dates when you have separate year, month, and day fields in your documents. Use `$dateToParts` to decompose existing dates into modifiable components. Together, they enable powerful date manipulation like rounding, bucketing, and rebuilding dates with modified components.
