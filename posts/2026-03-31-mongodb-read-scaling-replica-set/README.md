# How to Set Up a Replica Set for Read Scaling in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Read Preference, Performance, Scalability

Description: Learn how to configure a MongoDB replica set for read scaling by routing read operations to secondary members using read preferences and connection strings.

---

MongoDB replica sets are primarily for high availability, but they also enable read scaling. By directing read-heavy queries to secondary members, you can offload the primary and serve more concurrent read requests. This guide covers read preferences, connection string options, and the consistency trade-offs involved.

## Read Preferences

MongoDB supports five read preference modes:

| Mode | Description |
|---|---|
| `primary` | All reads go to primary (default) |
| `primaryPreferred` | Primary if available, else secondary |
| `secondary` | Always read from a secondary |
| `secondaryPreferred` | Secondary if available, else primary |
| `nearest` | Member with lowest network latency |

## Connection String Configuration

Set read preference in the connection URI:

```text
mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0&readPreference=secondaryPreferred
```

## Node.js with Mongoose

```javascript
const mongoose = require('mongoose');

mongoose.connect('mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0', {
  readPreference: 'secondaryPreferred'
});

// Per-query override
const results = await Product.find({ category: 'electronics' })
  .read('secondary');
```

## Python with PyMongo

```python
from pymongo import MongoClient
from pymongo.read_preferences import Secondary, SecondaryPreferred

client = MongoClient(
    "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0",
    read_preference=SecondaryPreferred()
)

# Per-collection override
reports = client.db.get_collection('reports', read_preference=Secondary())
```

## Java Driver

```java
MongoClientSettings settings = MongoClientSettings.builder()
    .applyConnectionString(new ConnectionString(
        "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0"
    ))
    .readPreference(ReadPreference.secondaryPreferred())
    .build();

MongoClient client = MongoClients.create(settings);
```

## Tag-Based Read Routing

For fine-grained control, tag secondaries with custom labels and route reads to specific members:

```javascript
// Set tags in replica set config
var cfg = rs.conf();
cfg.members[1].tags = { region: "us-east", purpose: "reporting" };
cfg.members[2].tags = { region: "us-west", purpose: "app" };
rs.reconfig(cfg);
```

Then route reads to the reporting secondary:

```javascript
// Node.js
const results = await Report.find({}).read('secondary', [{ purpose: 'reporting' }]);
```

## Dedicated Analytics Secondary

Add a secondary with `priority: 0` specifically for analytics workloads:

```javascript
var cfg = rs.conf();
cfg.members.push({
  _id: 3,
  host: "mongo-analytics:27017",
  priority: 0,
  votes: 0,
  tags: { purpose: "analytics" }
});
rs.reconfig(cfg);
```

Connect the analytics service with `readPreference=secondary` and tag `purpose: analytics`.

## Consistency Trade-offs

Secondary reads are **eventually consistent** - they may lag behind the primary by milliseconds to seconds. This is acceptable for:
- Reporting and dashboards
- Search autocomplete
- Cache population

It is NOT acceptable for:
- Reads that must immediately reflect a prior write from the same session (use `readPreference: primary` or causal consistency)
- Financial balance checks

## Summary

MongoDB replica sets scale reads by distributing read traffic across secondary members. Use `secondaryPreferred` for most read-heavy applications, `secondary` for dedicated analytics secondaries, and tag-based routing for fine-grained control. Always consider replication lag - secondary reads are eventually consistent, making them suitable for analytics and reporting but not for read-after-write scenarios.

