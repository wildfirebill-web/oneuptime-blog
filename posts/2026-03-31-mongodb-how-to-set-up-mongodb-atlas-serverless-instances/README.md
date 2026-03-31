# How to Set Up MongoDB Atlas Serverless Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Serverless, Scalability, Cost Optimization

Description: Create and configure MongoDB Atlas Serverless instances that scale automatically and charge only for actual usage, ideal for variable or unpredictable workloads.

---

## What Is a Serverless Instance?

Atlas Serverless instances automatically scale compute and storage without you choosing a cluster tier. You pay only for the operations you perform - measured in Read Processing Units (RPUs) and Write Processing Units (WPUs) - rather than paying for a fixed cluster size 24/7.

When to use serverless:
- Development and testing environments
- Applications with unpredictable or highly variable traffic
- Applications with many idle periods
- New projects where usage is unknown

When to use dedicated clusters instead:
- High-volume production workloads (dedicated is cheaper above a certain threshold)
- Applications requiring change streams, transactions, or Atlas Search
- Low-latency requirements (dedicated clusters have more predictable latency)

## Step 1: Create a Serverless Instance

In Atlas, click **Create** and select **Serverless** instead of Dedicated or Shared:

1. Choose a cloud provider (AWS, GCP, or Azure)
2. Select a region
3. Name the instance (e.g., `myServerless`)
4. Click **Create Instance**

Via Atlas CLI:

```bash
atlas serverless create myServerless \
  --provider AWS \
  --region US_EAST_1
```

Instance provisions in about 60 seconds.

## Step 2: Create a Database User

```bash
atlas dbusers create \
  --username myUser \
  --password "SecurePass123!" \
  --role readWriteAnyDatabase
```

## Step 3: Configure Network Access

```bash
# Allow specific IP
atlas accessLists create \
  --cidr "203.0.113.0/32" \
  --comment "App server IP"

# Or allow all (development only)
atlas accessLists create \
  --cidr "0.0.0.0/0" \
  --comment "Dev - allow all"
```

## Step 4: Get the Connection String

```bash
atlas serverless describe myServerless
```

Output:

```text
ID                       NAME           STATE
6548abc123def456789012   myServerless   IDLE

CONNECTION STRING:
mongodb+srv://myServerless.abc123.mongodb.net
```

Connect:

```bash
mongosh "mongodb+srv://myUser:SecurePass123!@myServerless.abc123.mongodb.net/?retryWrites=true&w=majority"
```

## Step 5: Connect with Node.js

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient(
  "mongodb+srv://myUser:SecurePass123!@myServerless.abc123.mongodb.net/?retryWrites=true&w=majority"
);

async function run() {
  await client.connect();
  const db = client.db("myapp");

  // Operations are billed per RPU/WPU
  await db.collection("users").insertOne({
    name: "Alice",
    email: "alice@example.com",
    createdAt: new Date()
  });

  const user = await db.collection("users").findOne({ email: "alice@example.com" });
  console.log(user);

  await client.close();
}

run();
```

## Step 6: Understand Billing

Serverless billing is based on:

| Operation | Unit | Approximate Cost |
|---|---|---|
| Read | 1 RPU per document read | $0.10 per million RPUs |
| Write | 1 WPU per document written | $1.00 per million WPUs |
| Storage | GB/month | $0.25/GB |

Monitor usage in Atlas under **Billing > Serverless Usage**.

## Step 7: Serverless Limitations

Important limitations to be aware of:

```text
- No change streams
- No Atlas Search
- No Atlas Vector Search
- No cross-collection transactions (single-document is supported)
- Maximum document size: 16MB (same as dedicated)
- No aggregation $out to Atlas cluster
- Connection limit: auto-managed (you cannot configure maxPoolSize)
```

## Step 8: Upgrade to Dedicated Cluster

When traffic grows and dedicated becomes cheaper, migrate:

```bash
# Take a snapshot of serverless data
mongodump --uri "mongodb+srv://myUser:pass@myServerless.abc123.mongodb.net" \
  --out ./backup

# Restore to new dedicated cluster
mongorestore --uri "mongodb+srv://myUser:pass@myDedicated.abc123.mongodb.net" \
  ./backup
```

Or use Atlas Live Migration for zero-downtime migration.

## Step 9: Pause and Resume

Serverless instances cannot be paused like shared clusters, but they incur no compute costs when idle (no operations running). Storage costs still apply.

To check current instance state:

```bash
atlas serverless describe myServerless
```

States: `IDLE`, `LOADING`, `CREATING`, `DELETED`

## Summary

Atlas Serverless instances provision in seconds, scale automatically, and bill based on actual operation count rather than a fixed cluster size. Create an instance, configure a database user and network access, connect with any standard MongoDB driver, and pay only for what you use. They're ideal for development and variable workloads, but evaluate dedicated clusters for high-throughput production use cases where predictable billing and full feature support are required.
