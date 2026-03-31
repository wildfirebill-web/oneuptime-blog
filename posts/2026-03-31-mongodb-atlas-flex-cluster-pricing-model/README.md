# How to Understand Atlas Flex Cluster Pricing Model

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Flex Cluster, Pricing, Cost

Description: Understand how MongoDB Atlas Flex cluster pricing works with storage, operation-based billing, and how to estimate and control costs for low-traffic workloads.

---

## Overview

Atlas Flex clusters use a consumption-based pricing model that charges separately for storage and database operations. Unlike dedicated clusters with fixed hourly rates, Flex pricing scales with actual usage - making it economical for workloads with low or sporadic traffic while remaining predictable with usage caps.

## Flex Pricing Components

Flex clusters charge for two primary dimensions:

1. **Storage** - charged per GB-hour for the data and index storage consumed
2. **Read/Write Processing Units (RPUs/WPUs)** - charged per million operations executed

```text
Storage:    ~$0.025 per GB-month (varies by region and cloud provider)
Reads (RPU): ~$0.10 per million reads
Writes (WPU): ~$1.00 per million writes
```

These rates are approximate and vary by region. Always check the current Atlas pricing page for exact figures.

## Estimating Monthly Cost

Estimate cost based on expected operations and storage.

```javascript
function estimateFlexCost({
  storageMB = 500,
  readsPerDay = 10000,
  writesPerDay = 2000,
  days = 30,
  storageRatePerGB = 0.025,
  readRatePerMillion = 0.10,
  writeRatePerMillion = 1.00,
}) {
  const storageGB = storageMB / 1024;
  const storageCost = storageGB * storageRatePerGB;

  const totalReads = readsPerDay * days;
  const totalWrites = writesPerDay * days;
  const readCost = (totalReads / 1_000_000) * readRatePerMillion;
  const writeCost = (totalWrites / 1_000_000) * writeRatePerMillion;

  const total = storageCost + readCost + writeCost;

  return {
    storageCost: storageCost.toFixed(4),
    readCost: readCost.toFixed(4),
    writeCost: writeCost.toFixed(4),
    totalMonthly: total.toFixed(2),
  };
}

console.log(estimateFlexCost({
  storageMB: 500,
  readsPerDay: 10000,
  writesPerDay: 2000,
}));
// { storageCost: '0.0122', readCost: '0.0300', writeCost: '0.0600', totalMonthly: '0.10' }
```

## Monitoring Actual Usage via the Atlas API

Pull current billing metrics programmatically to track spending.

```bash
# Get cost breakdown for the current billing period
curl -X GET \
  "https://cloud.mongodb.com/api/atlas/v2/orgs/${ORG_ID}/invoices/pending" \
  -H "Accept: application/vnd.atlas.2023-01-01+json" \
  --digest -u "${PUBLIC_KEY}:${PRIVATE_KEY}" | \
  jq '.lineItems[] | select(.clusterName == "my-flex-cluster") | {sku, quantity, totalPriceCents}'
```

## Cost-Reduction Strategies

**1. Minimize unnecessary writes**

Batch multiple small updates into a single `updateOne` call to reduce write processing units.

```javascript
// Instead of multiple updates:
// await col.updateOne({_id}, {$set: {a: 1}});
// await col.updateOne({_id}, {$set: {b: 2}});

// Batch into one write:
await col.updateOne({ _id: docId }, { $set: { a: 1, b: 2 } });
```

**2. Use projection to reduce read payload size**

```javascript
// Fetch only needed fields to reduce RPU consumption
await col.find({}, { projection: { _id: 1, name: 1, status: 1 } }).toArray();
```

**3. Use covered queries to avoid document fetches**

```javascript
// Create an index that covers the query
await col.createIndex({ status: 1, createdAt: -1, _id: 1 });

// Query only returns index fields
await col.find({ status: "active" }, { projection: { status: 1, createdAt: 1, _id: 1 } }).toArray();
```

## Flex vs Dedicated Cost Comparison

```text
Workload: 500 MB storage, 10K reads/day, 2K writes/day

Flex cluster:  ~$0.10/month
M10 dedicated: ~$57/month
M0 free:       $0 (limited to 512 MB, no SLA)

Flex is ideal when your workload outgrows M0 but doesn't justify M10.
```

## Summary

Atlas Flex cluster pricing charges for storage (per GB-month) and operations (per million reads and writes). For small applications with a few thousand daily operations and under 1 GB of data, monthly costs are typically under a dollar. Reduce costs by batching writes, using projections, and leveraging covered queries. Upgrade to a dedicated cluster when consistent throughput guarantees become necessary.
