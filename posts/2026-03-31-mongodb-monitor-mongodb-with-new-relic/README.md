# How to Monitor MongoDB with New Relic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, New Relic, Monitoring, APM, Observability

Description: Set up New Relic to monitor MongoDB with the on-host integration, APM agent tracing, and custom dashboards for database performance.

---

## Introduction

New Relic provides MongoDB monitoring through its on-host integration (OHI) and APM agents that auto-instrument MongoDB drivers. Together, these give you infrastructure-level metrics and application-level query traces in a unified platform.

## Installing the New Relic Infrastructure Agent

```bash
# Install New Relic Infrastructure agent (Ubuntu/Debian)
curl -Ls https://download.newrelic.com/install/newrelic-cli/scripts/install.sh | bash

sudo NEW_RELIC_API_KEY=<YOUR_API_KEY> \
     NEW_RELIC_ACCOUNT_ID=<YOUR_ACCOUNT_ID> \
     /usr/local/bin/newrelic install -n infrastructure-agent-installer
```

## Installing the MongoDB On-Host Integration

```bash
sudo apt-get install nri-mongodb
```

## Configuring the Integration

```yaml
# /etc/newrelic-infra/integrations.d/mongodb-config.yml
integrations:
  - name: nri-mongodb
    env:
      HOST: localhost
      PORT: 27017
      USERNAME: newrelic
      PASSWORD: newrelicPass
      AUTH_SOURCE: admin
      SSL: false
      CLUSTER_NAME: production-cluster
      METRICS: true
      INVENTORY: true
      EVENTS: true
    interval: 30s
```

## Creating a Monitor User

```javascript
use admin
db.createUser({
  user: "newrelic",
  pwd: "newrelicPass",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" },
    { role: "readAnyDatabase", db: "admin" }
  ]
})
```

## APM Integration with Node.js

```javascript
// newrelic.js - must be required first
"use strict"
require("newrelic")

// mongodb operations are auto-instrumented
const { MongoClient } = require("mongodb")

async function findUsers() {
  const client = new MongoClient(process.env.MONGODB_URI)
  const db = client.db("myapp")
  // New Relic traces this as a database segment automatically
  return db.collection("users").find({ active: true }).toArray()
}
```

## APM Integration with Python

```python
# newrelic.ini must be configured before starting the app
import newrelic.agent
newrelic.agent.initialize("newrelic.ini")

from pymongo import MongoClient

@newrelic.agent.function_trace()
def get_active_users():
    client = MongoClient(os.environ["MONGODB_URI"])
    # pymongo calls are auto-instrumented for New Relic APM
    return list(client.myapp.users.find({"active": True}))
```

## Querying MongoDB Metrics in NRQL

Once data is flowing, use NRQL to analyze MongoDB performance:

```sql
-- Check connection usage over time
SELECT average(mongodb.connections.current)
FROM Metric
WHERE clusterName = 'production-cluster'
TIMESERIES 5 minutes
SINCE 1 hour ago

-- Find slowest collections
SELECT average(mongodb.collection.avgObjSize)
FROM Metric
FACET collectionName
SINCE 1 hour ago

-- Replication lag
SELECT latest(mongodb.replset.lag)
FROM Metric
FACET memberName
TIMESERIES
```

## Creating an Alert Condition

```bash
# Using New Relic CLI
newrelic alerts conditions create \
  --policy-id <POLICY_ID> \
  --name "MongoDB High Connection Count" \
  --type static \
  --metric "mongodb.connections.current" \
  --threshold 800 \
  --threshold-duration 5 \
  --threshold-occurrences ALL
```

## Summary

New Relic MongoDB monitoring combines the on-host integration for infrastructure metrics with APM agents for query-level tracing. Install `nri-mongodb` alongside the Infrastructure agent, create a dedicated monitor user, and configure the integration YAML. Use NRQL to build custom dashboards and alert conditions for connection saturation and replication lag. The APM layer adds query timing and slow operation detection directly in your application traces.
