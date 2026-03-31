# MongoDB vs Cassandra: When to Choose Each

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Cassandra, NoSQL, Comparison, Database

Description: Compare MongoDB and Cassandra across data modeling, write performance, consistency models, and use cases to pick the right distributed database.

---

## Overview

MongoDB and Apache Cassandra are both popular NoSQL databases, but they are optimized for different workloads. MongoDB is a document database built for flexible queries and rich data models. Cassandra is a wide-column store optimized for high-throughput writes and linear horizontal scalability.

## Data Model

MongoDB stores JSON-like documents:

```javascript
db.sensor_readings.insertOne({
  sensorId: "sensor-42",
  timestamp: new Date(),
  temperature: 23.5,
  humidity: 60.2,
  metadata: { location: "warehouse-A", unit: "celsius" }
})
```

Cassandra uses a wide-column model where tables are designed around query patterns:

```sql
CREATE TABLE sensor_readings (
  sensor_id TEXT,
  bucket TEXT,
  timestamp TIMESTAMP,
  temperature DOUBLE,
  humidity DOUBLE,
  PRIMARY KEY ((sensor_id, bucket), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

## Write Performance

Cassandra is purpose-built for write-heavy workloads. Its log-structured storage (LSM trees) provides extremely fast writes with no read-before-write overhead. MongoDB's WiredTiger storage engine is write-efficient but has higher write amplification on updates.

## Consistency Model

Cassandra is tunable - you can choose eventual or strong consistency per operation:

```python
from cassandra.cluster import Cluster
from cassandra import ConsistencyLevel
from cassandra.query import SimpleStatement

cluster = Cluster(["cassandra-host"])
session = cluster.connect("mydb")

# Strong consistency for reads
stmt = SimpleStatement(
    "SELECT * FROM sensor_readings WHERE sensor_id = %s",
    consistency_level=ConsistencyLevel.QUORUM
)
rows = session.execute(stmt, ["sensor-42"])
```

MongoDB provides strong consistency on replica set primaries by default.

## Querying

MongoDB supports rich ad-hoc queries and aggregations. Cassandra restricts queries to the primary key and indexed columns to guarantee performance:

```sql
-- Cassandra: this is efficient
SELECT * FROM sensor_readings WHERE sensor_id = 'sensor-42' AND bucket = '2024-01';

-- Cassandra: this requires ALLOW FILTERING and scans the entire table
SELECT * FROM sensor_readings WHERE temperature > 30 ALLOW FILTERING;
```

## When to Choose Cassandra

- Time-series data (IoT sensor readings, metrics, logs)
- Write-heavy workloads exceeding millions of records per second
- Multi-datacenter active-active deployments requiring tunable consistency
- Applications with predictable, query-driven data access patterns

## When to Choose MongoDB

- Applications requiring flexible queries and aggregations without pre-planning
- Rich document data with nested structures and varying schemas
- Transactional workloads requiring multi-document ACID compliance
- Teams more familiar with JSON and document-oriented design

## Summary

Cassandra outperforms MongoDB on raw write throughput and linear horizontal scalability, making it the better choice for IoT, time-series, and write-dominant workloads. MongoDB offers richer querying, better support for evolving schemas, and full ACID transactions, making it the stronger choice for general-purpose application data. Evaluate your read-to-write ratio and query patterns before choosing.
