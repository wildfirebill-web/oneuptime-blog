# How to Use $isoDayOfWeek, $isoWeek, and $isoWeekYear in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Date, $isoWeek, $isoDayOfWeek

Description: Learn how to use $isoDayOfWeek, $isoWeek, and $isoWeekYear in MongoDB aggregation to work with ISO 8601 week-based calendars for accurate weekly reporting.

---

MongoDB's standard `$dayOfWeek` and `$week` operators follow a Sunday-based calendar. For business reporting that follows the ISO 8601 standard - where weeks start on Monday and Week 1 is the week containing the first Thursday of the year - you need the ISO-specific operators.

## The ISO Calendar Operators

| Operator | Returns |
|----------|---------|
| `$isoDayOfWeek` | Day of week 1 (Monday) to 7 (Sunday) |
| `$isoWeek` | ISO week number 1-53 |
| `$isoWeekYear` | ISO week-numbering year |

Note that `$isoWeekYear` can differ from `$year` at year boundaries. For example, January 1, 2026 falls in ISO week 1 of 2026, but dates in late December may belong to the next ISO year.

## Basic Usage

```js
db.orders.aggregate([
  {
    $project: {
      orderDate: "$createdAt",
      isoDay:       { $isoDayOfWeek: "$createdAt" },
      isoWeek:      { $isoWeek: "$createdAt" },
      isoWeekYear:  { $isoWeekYear: "$createdAt" }
    }
  }
]);
```

For a document with `createdAt: 2026-03-30T10:00:00Z` (a Monday):

```text
{ isoDay: 1, isoWeek: 14, isoWeekYear: 2026 }
```

## Grouping by ISO Week

Generate weekly revenue reports aligned to Monday-Sunday weeks:

```js
db.sales.aggregate([
  {
    $group: {
      _id: {
        year: { $isoWeekYear: "$saleDate" },
        week: { $isoWeek: "$saleDate" }
      },
      weeklyRevenue: { $sum: "$amount" },
      orderCount:    { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.week": 1 } }
]);
```

## Finding Day-of-Week Distribution (ISO)

Compare traffic across weekdays vs weekends using ISO day numbering where 1=Monday, 6=Saturday, 7=Sunday:

```js
db.sessions.aggregate([
  {
    $group: {
      _id: { $isoDayOfWeek: "$startedAt" },
      sessions: { $sum: 1 }
    }
  },
  {
    $project: {
      dayLabel: {
        $arrayElemAt: [
          ["", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"],
          "$_id"
        ]
      },
      sessions: 1
    }
  },
  { $sort: { _id: 1 } }
]);
```

## Year-Boundary Edge Case

Dates around December 31 and January 1 can have a different `$isoWeekYear` than `$year`. Always use `$isoWeekYear` with `$isoWeek`, not `$year`:

```js
db.orders.aggregate([
  {
    $project: {
      date: "$createdAt",
      // Correct: ISO year matches ISO week
      isoGroup: {
        year: { $isoWeekYear: "$createdAt" },
        week: { $isoWeek: "$createdAt" }
      },
      // Potentially wrong at year boundaries
      mixedGroup: {
        year: { $year: "$createdAt" },
        week: { $isoWeek: "$createdAt" }
      }
    }
  }
]);
```

## Timezone Support

Pass the date object form to include a timezone:

```js
db.orders.aggregate([
  {
    $project: {
      isoWeek: {
        $isoWeek: {
          date: "$createdAt",
          timezone: "Europe/London"
        }
      }
    }
  }
]);
```

## Filtering for Specific ISO Week

Find all records from ISO week 10 of 2026:

```js
db.orders.aggregate([
  {
    $match: {
      $expr: {
        $and: [
          { $eq: [{ $isoWeekYear: "$createdAt" }, 2026] },
          { $eq: [{ $isoWeek: "$createdAt" }, 10] }
        ]
      }
    }
  }
]);
```

## Summary

Use `$isoDayOfWeek`, `$isoWeek`, and `$isoWeekYear` whenever your reporting follows ISO 8601 week numbering, which starts weeks on Monday and assigns week numbers based on the week containing the first Thursday of the year. Always pair `$isoWeek` with `$isoWeekYear` rather than `$year` to avoid incorrect grouping at year boundaries.
