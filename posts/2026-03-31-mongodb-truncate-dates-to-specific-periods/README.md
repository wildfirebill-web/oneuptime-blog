# How to Truncate Dates to Specific Periods in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Date, Aggregation

Description: Learn how to truncate dates to hours, days, weeks, or months in MongoDB using $dateTrunc and $dateToString for time-series grouping.

---

Date truncation rounds a timestamp down to the start of a time period - the beginning of the hour, day, week, or month. This is essential for time-series aggregations where you want to group events by bucket rather than exact timestamp.

## Using $dateTrunc (MongoDB 5.0+)

The `$dateTrunc` operator is the cleanest way to truncate dates. It takes a date and a unit, and returns the start of that time period:

```javascript
db.metrics.aggregate([
  {
    $project: {
      hourBucket: {
        $dateTrunc: {
          date: "$recordedAt",
          unit: "hour"
        }
      },
      dayBucket: {
        $dateTrunc: {
          date: "$recordedAt",
          unit: "day"
        }
      },
      monthBucket: {
        $dateTrunc: {
          date: "$recordedAt",
          unit: "month"
        }
      }
    }
  }
]);
```

The `unit` field accepts: `"millisecond"`, `"second"`, `"minute"`, `"hour"`, `"day"`, `"week"`, `"month"`, `"quarter"`, `"year"`.

## Truncating with a Timezone

By default, `$dateTrunc` operates in UTC. For day or week buckets that should align with a user's local midnight, pass a `timezone`:

```javascript
db.metrics.aggregate([
  {
    $project: {
      localDayBucket: {
        $dateTrunc: {
          date: "$recordedAt",
          unit: "day",
          timezone: "America/New_York"
        }
      }
    }
  }
]);
```

The returned Date is still UTC-normalized, but it aligns with midnight Eastern time.

## Truncating with a Custom Bin Size

`$dateTrunc` supports a `binSize` parameter for multi-unit buckets. To group into 6-hour windows:

```javascript
db.metrics.aggregate([
  {
    $project: {
      sixHourBucket: {
        $dateTrunc: {
          date: "$recordedAt",
          unit: "hour",
          binSize: 6
        }
      }
    }
  }
]);
```

## Grouping Events by Time Bucket

The real power of date truncation is in `$group` stages. Here is a full pipeline that counts events per hour:

```javascript
db.events.aggregate([
  {
    $match: {
      occurredAt: {
        $gte: ISODate("2026-03-01T00:00:00Z"),
        $lt:  ISODate("2026-04-01T00:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: {
        $dateTrunc: {
          date: "$occurredAt",
          unit: "hour"
        }
      },
      count: { $sum: 1 }
    }
  },
  {
    $sort: { _id: 1 }
  }
]);
```

## Alternative: $dateToString for Older MongoDB Versions

Before MongoDB 5.0, use `$dateToString` to produce a string key for grouping. The truncation is implicit in the format string:

```javascript
db.events.aggregate([
  {
    $group: {
      _id: {
        $dateToString: {
          format: "%Y-%m-%dT%H:00:00Z",
          date: "$occurredAt"
        }
      },
      count: { $sum: 1 }
    }
  }
]);
```

The format `"%Y-%m-%dT%H:00:00Z"` zeroes out minutes and seconds, effectively truncating to the hour. For day buckets, use `"%Y-%m-%d"`.

## Week Truncation and startOfWeek

`$dateTrunc` with `unit: "week"` defaults to Sunday as the start of the week. Override this with `startOfWeek`:

```javascript
db.events.aggregate([
  {
    $project: {
      weekBucket: {
        $dateTrunc: {
          date: "$occurredAt",
          unit: "week",
          startOfWeek: "monday"
        }
      }
    }
  }
]);
```

## Summary

Use `$dateTrunc` (MongoDB 5.0+) to truncate dates to standard periods in aggregation pipelines. Supply a `timezone` when your buckets should align with local midnight. For multi-unit buckets like 15-minute windows, use `binSize`. On older MongoDB versions, `$dateToString` with a zeroed format string achieves the same grouping at the cost of returning strings instead of Date objects.
