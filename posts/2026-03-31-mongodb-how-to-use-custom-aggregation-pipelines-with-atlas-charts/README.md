# How to Use Custom Aggregation Pipelines with Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Aggregation, Data Visualization, Analytics

Description: Learn how to use custom aggregation pipelines as data sources in MongoDB Atlas Charts to transform, join, and compute complex metrics before visualization.

---

## Why Use Custom Pipelines in Atlas Charts

By default, Atlas Charts reads raw collection data and applies chart-level aggregations. Custom aggregation pipelines let you pre-transform data before it reaches the chart builder - enabling joins across collections, computed fields, time bucketing, and complex filtering that the chart builder alone cannot express.

## Configuring a Custom Pipeline Data Source

1. Open Atlas Charts and click **Add Chart** or edit an existing chart
2. In the data source panel, click **Data Source** dropdown
3. Select your cluster and database
4. Click the **Aggregation Pipeline** toggle to switch from collection to pipeline mode
5. Enter your pipeline in the editor
6. Click **Apply** to preview the transformed data

## Example 1 - Computing Revenue Metrics

Transform raw order data into pre-aggregated revenue metrics:

```javascript
[
  {
    $match: {
      status: { $in: ["completed", "shipped"] },
      orderDate: { $gte: ISODate("2026-01-01") }
    }
  },
  {
    $group: {
      _id: {
        year: { $year: "$orderDate" },
        month: { $month: "$orderDate" },
        region: "$region"
      },
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$amount" },
      uniqueCustomers: { $addToSet: "$customerId" }
    }
  },
  {
    $addFields: {
      uniqueCustomerCount: { $size: "$uniqueCustomers" },
      date: {
        $dateFromParts: {
          year: "$_id.year",
          month: "$_id.month"
        }
      }
    }
  },
  {
    $project: {
      uniqueCustomers: 0  // remove large array from output
    }
  },
  { $sort: { date: 1, "_id.region": 1 } }
]
```

This gives you a clean table with `date`, `region`, `totalRevenue`, `orderCount`, and `avgOrderValue` - ready to map to any chart type.

## Example 2 - Multi-Collection Join with $lookup

Join orders with customer data for enriched charts:

```javascript
[
  {
    $match: { orderDate: { $gte: ISODate("2026-01-01") } }
  },
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer",
      pipeline: [
        { $project: { name: 1, tier: 1, country: 1 } }
      ]
    }
  },
  { $unwind: { path: "$customer", preserveNullAndEmptyArrays: true } },
  {
    $group: {
      _id: { tier: "$customer.tier", month: { $month: "$orderDate" } },
      revenue: { $sum: "$amount" },
      orders: { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      tier: { $ifNull: ["$_id.tier", "Unknown"] },
      month: "$_id.month",
      revenue: 1,
      orders: 1
    }
  },
  { $sort: { month: 1, tier: 1 } }
]
```

## Example 3 - Cohort Analysis

Compute user cohorts for a retention chart:

```javascript
[
  {
    $group: {
      _id: "$userId",
      firstOrderDate: { $min: "$orderDate" },
      orderDates: { $push: "$orderDate" }
    }
  },
  {
    $addFields: {
      cohortMonth: {
        $dateToString: {
          format: "%Y-%m",
          date: "$firstOrderDate"
        }
      },
      orderMonths: {
        $map: {
          input: "$orderDates",
          as: "d",
          in: {
            $dateToString: { format: "%Y-%m", date: "$$d" }
          }
        }
      }
    }
  },
  {
    $group: {
      _id: "$cohortMonth",
      cohortSize: { $sum: 1 },
      month1Retained: {
        $sum: {
          $cond: [
            {
              $in: [
                {
                  $dateToString: {
                    format: "%Y-%m",
                    date: { $dateAdd: { startDate: { $dateFromString: { dateString: { $concat: ["$cohortMonth", "-01"] } } }, unit: "month", amount: 1 } }
                  }
                },
                "$orderMonths"
              ]
            },
            1, 0
          ]
        }
      }
    }
  },
  {
    $addFields: {
      retentionRate: {
        $multiply: [
          { $divide: ["$month1Retained", "$cohortSize"] },
          100
        ]
      }
    }
  },
  { $sort: { _id: 1 } }
]
```

## Example 4 - Time Series Bucketing

Create hourly buckets for a time series chart:

```javascript
[
  {
    $match: {
      timestamp: {
        $gte: ISODate("2026-03-31T00:00:00Z"),
        $lt:  ISODate("2026-04-01T00:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: {
        $dateTrunc: {
          date: "$timestamp",
          unit: "hour"
        }
      },
      avgValue: { $avg: "$value" },
      maxValue: { $max: "$value" },
      minValue: { $min: "$value" },
      count: { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      hour: "$_id",
      avgValue: { $round: ["$avgValue", 2] },
      maxValue: 1,
      minValue: 1,
      count: 1
    }
  },
  { $sort: { hour: 1 } }
]
```

## Example 5 - Funnel Stage Analysis

Compute funnel metrics for a funnel chart:

```javascript
[
  {
    $facet: {
      visited: [
        { $match: { event: "page_view" } },
        { $group: { _id: null, count: { $sum: 1 } } }
      ],
      addedToCart: [
        { $match: { event: "add_to_cart" } },
        { $group: { _id: null, count: { $sum: 1 } } }
      ],
      checkout: [
        { $match: { event: "checkout_start" } },
        { $group: { _id: null, count: { $sum: 1 } } }
      ],
      purchased: [
        { $match: { event: "purchase" } },
        { $group: { _id: null, count: { $sum: 1 } } }
      ]
    }
  },
  {
    $project: {
      stages: [
        { name: "Visited",      count: { $arrayElemAt: ["$visited.count",     0] } },
        { name: "Add to Cart",  count: { $arrayElemAt: ["$addedToCart.count", 0] } },
        { name: "Checkout",     count: { $arrayElemAt: ["$checkout.count",    0] } },
        { name: "Purchased",    count: { $arrayElemAt: ["$purchased.count",   0] } }
      ]
    }
  },
  { $unwind: "$stages" },
  { $replaceRoot: { newRoot: "$stages" } }
]
```

## Tips for Custom Pipelines in Atlas Charts

```text
1. Put $match first to leverage indexes and reduce pipeline input
2. Use $project early to drop fields you don't need
3. Avoid $unwind on large arrays without first $limiting input
4. Use $dateTrunc or $dateFromParts to normalize dates for time series charts
5. Test your pipeline in MongoDB Compass before pasting into Charts
6. Keep output field names simple - they become your chart encoding channel labels
```

## Summary

Custom aggregation pipelines in Atlas Charts unlock the full power of MongoDB's aggregation framework for visualization - enabling multi-collection joins with `$lookup`, time-series bucketing with `$dateTrunc`, funnel analysis with `$facet`, and cohort retention calculations. By transforming data before it reaches the chart builder, you get cleaner encoding options, faster chart rendering from pre-aggregated results, and the ability to express complex business logic that the chart builder's built-in aggregations cannot handle.
