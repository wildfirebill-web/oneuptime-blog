# How to Use MongoDB Atlas for Serverless Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Serverless, Cloud, Database

Description: Learn how to create and use MongoDB Atlas Serverless instances that scale to zero, charge per operation, and require no cluster capacity planning.

---

## What are Atlas Serverless Instances

MongoDB Atlas Serverless instances are database deployments that automatically scale up and down based on workload, including scaling to zero when idle. Unlike dedicated clusters where you pay for provisioned capacity, serverless instances charge per read, write, and storage unit consumed. This makes them ideal for applications with unpredictable or spiky traffic patterns.

```mermaid
flowchart LR
    App[Application] -->|Variable load| Serverless[Atlas Serverless Instance]
    Serverless -->|Scale to zero| Idle[No cost when idle]
    Serverless -->|Auto-scale up| ActiveLoad[Handle traffic spikes]
    Serverless -->|Per-operation billing| Cost[Pay only for what you use]
```

## When to Use Serverless Instances

Use serverless instances for:

- Development and staging environments with intermittent usage
- Applications with unpredictable traffic patterns (e.g., event-driven, seasonal)
- New projects where query volume is uncertain
- Microservices with low request rates that do not justify a dedicated cluster

Use dedicated clusters for:

- High-throughput production workloads with consistent load
- Applications requiring replica set reads from secondaries
- Workloads needing Atlas Search, Atlas Data Federation, or change streams at scale
- Latency-sensitive applications that cannot tolerate cold start delays

## Creating a Serverless Instance

Via the Atlas UI:

1. Log in to [cloud.mongodb.com](https://cloud.mongodb.com).
2. Click **Create** (or **+ New Cluster**).
3. Select **Serverless** as the deployment type.
4. Choose a cloud provider (AWS, GCP, or Azure) and region.
5. Name the instance (e.g., `my-serverless-db`).
6. Click **Create Instance**.

Via the Atlas CLI:

```bash
atlas serverless create my-serverless-db \
  --provider AWS \
  --region US_EAST_1
```

Via Terraform (using the MongoDB Atlas provider):

```javascript
resource "mongodbatlas_serverless_instance" "main" {
  project_id = var.project_id
  name       = "my-serverless-db"

  provider_settings_backing_provider_name = "AWS"
  provider_settings_provider_name         = "SERVERLESS"
  provider_settings_region_name           = "US_EAST_1"
}
```

## Connecting to a Serverless Instance

The connection string format is identical to a standard Atlas cluster:

```bash
mongodb+srv://<username>:<password>@my-serverless-db.abcde.mongodb.net/?retryWrites=true&w=majority
```

Connect with mongosh:

```bash
mongosh "mongodb+srv://appuser:password@my-serverless-db.abcde.mongodb.net/myapp"
```

Connect with the Node.js driver:

```javascript
const { MongoClient } = require("mongodb");

const uri = process.env.MONGODB_URI;
const client = new MongoClient(uri, {
  serverSelectionTimeoutMS: 10000,
  retryWrites: true
});

async function main() {
  await client.connect();
  const db = client.db("myapp");
  const result = await db.collection("orders").findOne({ status: "pending" });
  console.log(result);
  await client.close();
}

main();
```

## Inserting and Querying Data

All standard MongoDB operations work on serverless instances:

```javascript
const db = client.db("ecommerce");

// Insert a document
await db.collection("products").insertOne({
  name: "Wireless Headphones",
  price: 79.99,
  category: "electronics",
  stock: 150,
  createdAt: new Date()
});

// Find with projection
const products = await db.collection("products")
  .find({ category: "electronics", price: { $lt: 100 } })
  .project({ name: 1, price: 1 })
  .sort({ price: 1 })
  .toArray();

// Count documents
const count = await db.collection("products").countDocuments({ stock: { $gt: 0 } });
```

## Indexing on Serverless Instances

Indexes work the same way on serverless instances. Create them before loading production data:

```javascript
// Create compound index for common query patterns
await db.collection("orders").createIndex(
  { customerId: 1, createdAt: -1 },
  { name: "idx_orders_customer_date" }
);

// Create TTL index to expire sessions
await db.collection("sessions").createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }
);
```

Missing indexes on serverless instances are expensive because collection scans consume more read processing units (RPUs).

## Understanding the Billing Model

Atlas Serverless bills on:

- **Read Processing Units (RPUs)**: one RPU per batch of documents read
- **Write Processing Units (WPUs)**: one WPU per document written
- **Storage**: GB of data stored per hour
- **Data transfer**: outbound data transfer beyond the free tier

To minimize cost:

- Use indexes to reduce documents scanned per query
- Use projections to return only necessary fields
- Batch writes with `insertMany` instead of individual `insertOne` calls
- Avoid large unbounded `find()` scans

## Limitations of Serverless Instances

| Feature | Serverless | Dedicated Cluster |
|---|---|---|
| Atlas Search | Not supported | Supported |
| Change streams | Not supported | Supported |
| Data Federation | Not supported | Supported |
| Analytics nodes | Not supported | Supported |
| Multi-region | Not supported | Supported |
| Secondary reads | Not supported | Supported |
| Transactions | Supported | Supported |
| Aggregation | Supported | Supported |

## Cold Start Behavior

Serverless instances may have a cold start delay when the instance has been idle. The first connection attempt after a period of inactivity can take a few extra seconds. Design applications to handle this with appropriate timeout settings:

```javascript
const client = new MongoClient(uri, {
  serverSelectionTimeoutMS: 15000, // allow 15s for cold start
  connectTimeoutMS: 15000
});
```

## Monitoring Usage

In the Atlas UI, navigate to your serverless instance and click the **Metrics** tab. You can view:

- RPUs and WPUs consumed over time
- Storage usage
- Connection count

Use Atlas Data Explorer to browse collections without consuming RPUs (the UI uses internal access).

## Summary

MongoDB Atlas Serverless instances provide a consumption-based deployment with automatic scaling including scale-to-zero. They are ideal for development environments and applications with unpredictable traffic. Connect using the standard Atlas connection string, use indexes diligently to minimize RPU consumption, and be aware that features like Atlas Search and change streams are not available on serverless. For high-throughput or feature-rich production workloads, use a dedicated cluster instead.
