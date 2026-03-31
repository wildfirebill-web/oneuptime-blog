# How to Query Across Clusters and S3 in a Single Federated Query in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Data Federation, Data Architecture

Description: Use MongoDB Atlas Data Federation to run a single aggregation pipeline that joins live Atlas cluster data with historical S3 data for unified analytics queries.

---

Atlas Data Federation lets you combine multiple Atlas clusters and S3 data stores in one Federated Database Instance. A single aggregation pipeline can then reference collections from any of these sources, enabling joins between live operational data and historical archived data.

## Use Case

A common pattern is "hot-warm-cold" data tiering:

- **Hot (Atlas cluster)** - current 90 days of data, optimized for writes and real-time queries
- **Warm (Atlas cluster 2)** - 90-365 days, read-heavy replica
- **Cold (S3)** - 1+ year archive, stored as JSON or Parquet

A single federated query can span all three tiers.

## Storage Configuration

Map three data sources to virtual collections:

```json
{
  "databases": [
    {
      "name": "orders_unified",
      "collections": [
        {
          "name": "orders_recent",
          "dataSources": [
            {
              "storeName": "prod-cluster",
              "database": "production",
              "collection": "orders"
            }
          ]
        },
        {
          "name": "orders_historical",
          "dataSources": [
            {
              "storeName": "analytics-cluster",
              "database": "analytics",
              "collection": "orders_2023"
            }
          ]
        },
        {
          "name": "orders_archive",
          "dataSources": [
            {
              "storeName": "s3-archive",
              "path": "/orders/archive/{year}/{month}/*.json",
              "defaultFormat": ".json"
            }
          ]
        }
      ]
    }
  ],
  "stores": [
    {
      "name": "prod-cluster",
      "provider": "atlas",
      "clusterName": "prod-cluster",
      "projectId": "<project-id>"
    },
    {
      "name": "analytics-cluster",
      "provider": "atlas",
      "clusterName": "analytics-cluster",
      "projectId": "<project-id>"
    },
    {
      "name": "s3-archive",
      "provider": "s3",
      "region": "us-east-1",
      "bucket": "orders-archive"
    }
  ]
}
```

## Cross-Source Union Query

Use `$unionWith` to combine results from all three sources:

```javascript
// Connect to FDI
db = client.getDatabase("orders_unified");

db.orders_recent.aggregate([
  // Recent data from Atlas cluster
  { $match: { customerId: "cust_12345" } },
  { $project: { orderId: 1, amount: 1, date: 1, source: { $literal: "recent" } } },

  // Union with historical Atlas data
  {
    $unionWith: {
      coll: "orders_historical",
      pipeline: [
        { $match: { customerId: "cust_12345" } },
        { $project: { orderId: 1, amount: 1, date: 1, source: { $literal: "historical" } } }
      ]
    }
  },

  // Union with S3 archive
  {
    $unionWith: {
      coll: "orders_archive",
      pipeline: [
        { $match: { customerId: "cust_12345" } },
        { $project: { orderId: 1, amount: 1, date: 1, source: { $literal: "archive" } } }
      ]
    }
  },

  // Aggregate across all sources
  { $group: {
    _id: null,
    totalSpend: { $sum: "$amount" },
    orderCount: { $sum: 1 }
  }}
])
```

## Joining Live Customers with Archived Orders

```javascript
// Get top customers from S3 archive joined with current Atlas data
db.orders_archive.aggregate([
  {
    $match: {
      year: "2022",
      status: "completed"
    }
  },
  {
    $group: {
      _id: "$customerId",
      archiveRevenue: { $sum: "$amount" }
    }
  },
  {
    $lookup: {
      from: "orders_recent",    // Live Atlas collection
      localField: "_id",
      foreignField: "customerId",
      as: "recentOrders"
    }
  },
  {
    $addFields: {
      recentRevenue: { $sum: "$recentOrders.amount" },
      totalRevenue: { $add: ["$archiveRevenue", { $sum: "$recentOrders.amount" }] }
    }
  },
  { $sort: { totalRevenue: -1 } },
  { $limit: 100 }
])
```

## Performance Considerations

S3 queries are slower than Atlas cluster queries. Structure pipelines to filter S3 data using partition fields early:

```javascript
// Good - partition filter pushdown on S3
{ $match: { year: "2022", month: { $in: ["10", "11", "12"] } } }

// Bad - full S3 scan then filter
{ $match: { amount: { $gt: 1000 } } }
```

Apply expensive operations like `$lookup` against Atlas sources, not S3.

## Summary

Atlas Data Federation federated queries span multiple Atlas clusters and S3 using `$unionWith` to merge collections from different stores in a single aggregation pipeline. Map each data source in the FDI storage configuration, connect via the FDI connection string, and design pipelines to push filters into S3 partition paths early for acceptable performance on large archives.
