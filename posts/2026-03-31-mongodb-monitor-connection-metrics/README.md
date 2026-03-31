# How to Monitor MongoDB Connection Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Monitoring, Connection, Performance, Metrics

Description: Learn how to monitor MongoDB connection metrics including current connections, available connections, and connection pool utilization to detect connection exhaustion.

---

MongoDB connection metrics are critical for detecting connection pool exhaustion, driver misconfigurations, and scaling bottlenecks. The `connections` section of `serverStatus` exposes current, available, and total connections, giving visibility into how much connection capacity remains before new clients are refused.

## Reading Connection Metrics

```javascript
db.adminCommand({ serverStatus: 1 }).connections
```

Output:

```json
{
  "current": 45,
  "available": 774955,
  "totalCreated": 1203,
  "active": 12,
  "threaded": 45,
  "exhaustIsMaster": 3,
  "exhaustHello": 3,
  "awaitingTopologyChanges": 1
}
```

## Key Metrics Explained

```text
current         - Number of open connections right now
available       - Remaining connections before hitting the limit
totalCreated    - Cumulative connections created since startup
active          - Connections currently executing an operation
exhaustIsMaster - Connections using exhaust isMaster protocol (monitoring)
```

The effective connection limit is `current + available`. MongoDB defaults to 1,000,000 maximum connections on Linux (adjustable via `net.maxIncomingConnections`).

## Monitoring Connection Usage in Python

```python
import pymongo
import time

client = pymongo.MongoClient("mongodb://localhost:27017")

def get_connection_metrics():
    status = client.admin.command("serverStatus")
    conns = status["connections"]
    return {
        "current": conns["current"],
        "available": conns["available"],
        "active": conns.get("active", 0),
        "total_created": conns["totalCreated"],
        "utilization_pct": round(
            conns["current"] / (conns["current"] + conns["available"]) * 100, 2
        )
    }

while True:
    metrics = get_connection_metrics()
    print(
        f"Connections: current={metrics['current']}  "
        f"active={metrics['active']}  "
        f"available={metrics['available']}  "
        f"utilization={metrics['utilization_pct']}%"
    )
    time.sleep(10)
```

## Checking the Connection Limit

```javascript
db.adminCommand({ getCmdLineOpts: 1 })
// Look for: net.maxIncomingConnections
```

Or from the shell:

```bash
mongod --maxConns 5000
```

## Configuring the Connection Limit in mongod.conf

```yaml
net:
  maxIncomingConnections: 5000
```

## Diagnosing Connection Leaks

When `current` grows continuously without `active` keeping pace, you likely have a connection leak - clients opening connections without closing them:

```javascript
// List current connections with client info
db.adminCommand({ currentOp: 1, $all: true }).inprog.forEach(op => {
  if (op.client) print(op.client, op.appName, op.connectionId)
})
```

Group by application name to identify the leaking service:

```javascript
db.adminCommand({ currentOp: 1, $all: true }).inprog.reduce((acc, op) => {
  const app = op.appName || "unknown";
  acc[app] = (acc[app] || 0) + 1;
  return acc;
}, {})
```

## Alert Thresholds

Set alerts before connections are exhausted:

```python
metrics = get_connection_metrics()

if metrics["utilization_pct"] > 80:
    print(f"WARNING: Connection utilization at {metrics['utilization_pct']}%")

if metrics["available"] < 100:
    print(f"CRITICAL: Only {metrics['available']} connections remaining")
```

## Connection Metrics in Prometheus

The MongoDB Exporter exposes connection metrics:

```text
# Current connections gauge
mongodb_connections{state="current"}

# Available connections gauge
mongodb_connections{state="available"}

# Alert rule: connection utilization > 80%
- alert: MongoDBHighConnectionUsage
  expr: mongodb_connections{state="current"} /
        (mongodb_connections{state="current"} + mongodb_connections{state="available"}) > 0.8
  for: 5m
```

## Summary

MongoDB connection metrics from `serverStatus().connections` reveal how much connection capacity is in use. Monitor `current`, `available`, and `active` connections to detect connection pool exhaustion before it causes `TooManyConnections` errors. Set alerts at 80% utilization to give time for investigation. When `current` grows continuously, use `currentOp` to identify which applications are leaking connections and ensure connection pools are properly sized and closed when no longer needed.
