# MongoDB vs Cassandra: When to Choose Each

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Cassandra, Database Comparison, NoSQL, Architecture

Description: Compare MongoDB and Cassandra across data model, consistency, scaling, and query patterns to help you choose the right database for your use case.

---

## Overview

MongoDB and Cassandra are both popular NoSQL databases, but they are designed for fundamentally different workloads. MongoDB is a document database optimized for flexible queries and rich data models, while Cassandra is a wide-column store optimized for high-throughput writes and linear horizontal scalability.

## Data Model

### MongoDB

Documents stored as BSON (Binary JSON). Supports nested objects, arrays, and dynamic schemas:

```javascript
// MongoDB document
{
  "_id": ObjectId("..."),
  "userId": "u123",
  "profile": {
    "name": "Alice",
    "preferences": ["dark-mode", "notifications"]
  },
  "orders": [
    { "orderId": "o1", "amount": 99.99, "status": "shipped" }
  ]
}
```

### Cassandra

Tables with rows and columns. Data is denormalized by query pattern; each table is designed for a specific access pattern:

```cql
-- Cassandra table designed for queries by userId
CREATE TABLE orders_by_user (
  user_id TEXT,
  order_id UUID,
  amount DECIMAL,
  status TEXT,
  created_at TIMESTAMP,
  PRIMARY KEY (user_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

## Consistency Model

| Feature | MongoDB | Cassandra |
|---|---|---|
| Default | Strong (primary reads) | Eventual (tunable) |
| Transactions | Multi-document ACID (4.0+) | Lightweight transactions (LWT) only |
| Consistency levels | Majority, local, linearizable | ONE, QUORUM, ALL, LOCAL_QUORUM |

## Write Performance

Cassandra is purpose-built for high-throughput writes. Its log-structured storage (LSM tree) makes writes extremely fast:

- Cassandra: can handle millions of writes per second across a cluster
- MongoDB: strong write performance but writes go through a write concern acknowledgment path

For pure write-heavy workloads (IoT sensor data, event logging, time series), Cassandra has the edge.

## Query Flexibility

MongoDB supports rich, ad-hoc queries, aggregation pipelines, and secondary indexes:

```javascript
// MongoDB: complex ad-hoc query
db.orders.find({
  "shipping.address.state": "CA",
  amount: { $gt: 100 },
  tags: { $in: ["priority", "express"] }
}).sort({ createdAt: -1 }).limit(20);
```

Cassandra queries must align with table partition key and clustering key design:

```cql
-- Cassandra: must query by partition key (user_id)
SELECT * FROM orders_by_user
WHERE user_id = 'u123'
AND created_at > '2026-01-01'
LIMIT 20;
```

## Horizontal Scaling

Both scale horizontally, but differently:

- **Cassandra**: masterless ring topology, automatically distributes data. Scales linearly - add nodes, get proportional throughput gains.
- **MongoDB**: sharded cluster with a config server and mongos routers. Effective but more operationally complex.

## Use Cases

### Choose MongoDB when:

- You need flexible, ad-hoc queries and rich aggregations
- Your data model evolves frequently (schema changes)
- You need ACID transactions across multiple documents
- You are building a general-purpose application (CMS, e-commerce, user profiles)
- Your team is more comfortable with JSON-style data

### Choose Cassandra when:

- You need extreme write throughput (millions of events per second)
- Your queries are predictable and access patterns are known upfront
- You need multi-datacenter, active-active replication with zero downtime
- Your use case is time-series data, IoT, or messaging
- Linear horizontal scalability is a hard requirement

## Operational Comparison

| Aspect | MongoDB | Cassandra |
|---|---|---|
| Management | MongoDB Atlas (fully managed) or self-hosted | DataStax Astra or self-hosted |
| Schema changes | No downtime, flexible | Requires table redesign for query changes |
| Index types | Many (text, geo, compound, sparse) | Limited (secondary indexes are anti-pattern) |
| Backup | Mongodump, Atlas automated backups | nodetool snapshot |

## Hybrid Approach

Some architectures use both: MongoDB for the application's operational data store (ODS), and Cassandra for high-volume event streams and time-series data:

```text
User-facing API -> MongoDB (profiles, orders, inventory)
Event pipeline -> Cassandra (click events, sensor readings, audit logs)
```

## Summary

MongoDB excels at flexible data modeling, rich querying, and developer productivity, making it ideal for general-purpose applications. Cassandra excels at extreme write throughput, linear horizontal scaling, and multi-datacenter active-active deployments, making it ideal for time-series, IoT, and messaging workloads. If you need ad-hoc queries and evolving schemas, choose MongoDB. If you need predictable, massive write throughput with known access patterns, choose Cassandra.
