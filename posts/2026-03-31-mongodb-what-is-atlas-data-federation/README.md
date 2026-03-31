# What Is MongoDB Atlas Data Federation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Data Federation, Data Lake, S3, Federated Query

Description: MongoDB Atlas Data Federation lets you query data across Atlas clusters, S3 buckets, and HTTP endpoints using standard MongoDB query syntax from a single interface.

---

## Overview

MongoDB Atlas Data Federation is a query engine that enables you to run MongoDB queries across data stored in multiple locations - Atlas clusters, Amazon S3 buckets, Azure Blob Storage, Google Cloud Storage, and HTTP endpoints - without moving the data. You define a Federated Database Instance (FDI) that maps virtual collections to physical data sources, and queries run against the virtual collections as if they were regular MongoDB collections.

This eliminates the need to ETL data into a single location for cross-source analytics. You pay only for the data scanned, not for storing an additional copy.

## Key Use Cases

- **Archival queries** - Store old data in S3 at low cost and query it alongside live Atlas data
- **Data lake analytics** - Query Parquet or JSON files in S3 using MongoDB aggregation syntax
- **Cross-cluster queries** - Join data from multiple Atlas clusters in a single query
- **Log analysis** - Query archived log files stored in object storage
- **Compliance reporting** - Access historical data without loading it into the operational database

## How It Works

1. You create a Federated Database Instance in Atlas and define a "storage configuration" that maps data sources to virtual collection paths.
2. When a query hits the FDI endpoint, the Data Federation engine pushes down filters and projections to minimize data scanned.
3. Results are returned in MongoDB wire protocol format, compatible with any MongoDB driver.

## Example Storage Configuration

```json
{
  "stores": [
    {
      "name": "s3Store",
      "provider": "s3",
      "region": "us-east-1",
      "bucket": "my-data-bucket",
      "delimiter": "/"
    },
    {
      "name": "atlasStore",
      "provider": "atlas",
      "clusterName": "myCluster",
      "projectId": "<projectId>"
    }
  ],
  "databases": [
    {
      "name": "analyticsDB",
      "collections": [
        {
          "name": "archiveOrders",
          "dataSources": [
            {
              "storeName": "s3Store",
              "path": "/orders/archive/{year string}/{month string}/*.json",
              "defaultFormat": ".json"
            }
          ]
        },
        {
          "name": "liveOrders",
          "dataSources": [
            {
              "storeName": "atlasStore",
              "database": "shop",
              "collection": "orders"
            }
          ]
        }
      ]
    }
  ]
}
```

## Querying Federated Data

Connect using a standard MongoDB connection string for your FDI and query as normal:

```javascript
// Connect to the Federated Database Instance
const client = new MongoClient(federatedConnectionString);
const db = client.db("analyticsDB");

// Query archived data from S3
const archivedOrders = await db.collection("archiveOrders")
  .find({ year: "2024", status: "completed" })
  .toArray();

// Run aggregation across S3 data
const revenueByMonth = await db.collection("archiveOrders").aggregate([
  { $group: { _id: "$month", total: { $sum: "$amount" } } },
  { $sort: { _id: 1 } }
]).toArray();
```

## Supported File Formats in Object Storage

- JSON (`.json`)
- BSON (`.bson`)
- CSV (`.csv`)
- TSV (`.tsv`)
- Parquet (`.parquet`)
- Avro (`.avro`)

## Wildcard Collections

Data Federation supports wildcard collection names to dynamically map partitioned data:

```json
{
  "name": "*",
  "dataSources": [{
    "storeName": "s3Store",
    "path": "/logs/{collectionName string}/*.json"
  }]
}
```

This creates a virtual collection for each folder in the S3 path.

## Cost Considerations

Atlas Data Federation charges based on data processed (bytes scanned). To minimize cost:
- Store data in Parquet format (columnar, compressed)
- Use partition paths in your data so Federation can skip irrelevant files
- Add filters early in queries to reduce scanned data

## Summary

MongoDB Atlas Data Federation is a serverless query engine that lets you run MongoDB queries across live Atlas data, archived S3 data, and other sources without moving data. It is ideal for analytics, archival queries, and cost-effective data lake access using familiar MongoDB syntax.
