# How to Calculate Conversion Rates in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Conversion, Metric

Description: Learn how to calculate conversion rates in MongoDB by comparing users who completed a goal against those who started the process using aggregation.

---

## Introduction

A conversion rate measures the percentage of users who completed a desired action out of those who started the process. Common examples include signup-to-purchase conversion or trial-to-paid conversion. MongoDB's aggregation pipeline computes these rates by counting distinct users at each step.

## Sample Data

Assume a `user_events` collection:

```json
{
  "_id": ObjectId("..."),
  "userId": "user_500",
  "event": "trial_started",
  "timestamp": ISODate("2025-09-01T08:00:00Z")
}
```

Events of interest: `trial_started`, `onboarding_completed`, `paid_subscription`.

## Overall Conversion Rate

Calculate conversion from trial to paid in a single pipeline using `$facet`:

```javascript
db.user_events.aggregate([
  {
    $match: {
      timestamp: {
        $gte: ISODate("2025-09-01T00:00:00Z"),
        $lt: ISODate("2025-10-01T00:00:00Z")
      }
    }
  },
  {
    $facet: {
      trialists: [
        { $match: { event: "trial_started" } },
        { $group: { _id: null, users: { $addToSet: "$userId" } } },
        { $project: { count: { $size: "$users" } } }
      ],
      converted: [
        { $match: { event: "paid_subscription" } },
        { $group: { _id: null, users: { $addToSet: "$userId" } } },
        { $project: { count: { $size: "$users" } } }
      ]
    }
  },
  {
    $project: {
      trialists: { $arrayElemAt: ["$trialists.count", 0] },
      converted: { $arrayElemAt: ["$converted.count", 0] }
    }
  },
  {
    $project: {
      trialists: 1,
      converted: 1,
      conversionRate: {
        $cond: {
          if: { $gt: ["$trialists", 0] },
          then: { $multiply: [{ $divide: ["$converted", "$trialists"] }, 100] },
          else: 0
        }
      }
    }
  }
])
```

## Conversion Rate by Acquisition Channel

Break down conversion by the channel that brought the user in:

```javascript
db.user_events.aggregate([
  {
    $match: {
      event: { $in: ["trial_started", "paid_subscription"] }
    }
  },
  {
    $group: {
      _id: { channel: "$channel", event: "$event" },
      users: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      channel: "$_id.channel",
      event: "$_id.event",
      count: { $size: "$users" }
    }
  },
  {
    $group: {
      _id: "$channel",
      events: { $push: { event: "$event", count: "$count" } }
    }
  },
  {
    $project: {
      channel: "$_id",
      trialists: {
        $reduce: {
          input: "$events",
          initialValue: 0,
          in: {
            $cond: [
              { $eq: ["$$this.event", "trial_started"] },
              { $add: ["$$value", "$$this.count"] },
              "$$value"
            ]
          }
        }
      },
      converted: {
        $reduce: {
          input: "$events",
          initialValue: 0,
          in: {
            $cond: [
              { $eq: ["$$this.event", "paid_subscription"] },
              { $add: ["$$value", "$$this.count"] },
              "$$value"
            ]
          }
        }
      }
    }
  },
  {
    $project: {
      channel: 1,
      trialists: 1,
      converted: 1,
      conversionRate: {
        $multiply: [{ $divide: ["$converted", { $max: ["$trialists", 1] }] }, 100]
      }
    }
  },
  { $sort: { conversionRate: -1 } }
])
```

## Time-Series Conversion Trend

Track weekly conversion rates over time:

```javascript
db.user_events.aggregate([
  {
    $match: { event: { $in: ["trial_started", "paid_subscription"] } }
  },
  {
    $group: {
      _id: {
        week: { $dateTrunc: { date: "$timestamp", unit: "week" } },
        event: "$event"
      },
      users: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      week: "$_id.week",
      event: "$_id.event",
      count: { $size: "$users" }
    }
  }
])
```

## Index Recommendation

```javascript
db.user_events.createIndex({ event: 1, timestamp: 1, userId: 1 })
db.user_events.createIndex({ channel: 1, event: 1 })
```

## Summary

Calculating conversion rates in MongoDB uses `$facet` to count users at each step in a single pass, then `$divide` and `$multiply` to compute the percentage. For channel breakdowns, group by channel and event type, then use `$reduce` to extract counts per event. Always protect division with `$cond` or `$max` to avoid divide-by-zero errors. Index on `event`, `timestamp`, and `userId` for best aggregation performance.
