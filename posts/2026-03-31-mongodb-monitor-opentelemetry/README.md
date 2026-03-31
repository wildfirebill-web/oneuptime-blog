# How to Monitor MongoDB with OpenTelemetry

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, OpenTelemetry, Monitoring, Metrics, Trace

Description: Learn how to collect MongoDB metrics and traces using the OpenTelemetry Collector MongoDB receiver and instrument your application with auto-instrumentation.

---

## Overview

OpenTelemetry provides a vendor-neutral way to collect metrics, traces, and logs from MongoDB and your application. The OpenTelemetry Collector includes a MongoDB receiver that scrapes server stats, while language SDKs can instrument MongoDB driver calls automatically.

## Setting Up the OpenTelemetry Collector MongoDB Receiver

Add the MongoDB receiver to your collector config:

```yaml
receivers:
  mongodb:
    hosts:
      - endpoint: localhost:27017
    username: otel-monitor
    password: monitorpassword
    collection_interval: 60s
    tls:
      insecure: true

processors:
  batch:
    timeout: 10s

exporters:
  otlp:
    endpoint: "http://otel-backend:4317"

service:
  pipelines:
    metrics:
      receivers: [mongodb]
      processors: [batch]
      exporters: [otlp]
```

## Creating a MongoDB Monitoring User

```javascript
db.createUser({
  user: "otel-monitor",
  pwd: "monitorpassword",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }
  ]
})
```

## Key Metrics Collected

The MongoDB receiver collects:

- `mongodb.connections.count` - active, available, and current connections
- `mongodb.operations.count` - inserts, queries, updates, deletes per second
- `mongodb.memory.usage` - resident, virtual, and mapped memory
- `mongodb.document.count` - documents inserted, deleted, returned, and updated
- `mongodb.index.count` - number of indexes per collection
- `mongodb.cache.operations` - WiredTiger cache hits and misses

## Instrumenting Node.js Applications

The OpenTelemetry Node.js auto-instrumentation package automatically traces MongoDB driver calls:

```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node
```

```javascript
// tracing.js - load before your application code
const { NodeSDK } = require("@opentelemetry/sdk-node");
const { getNodeAutoInstrumentations } = require("@opentelemetry/auto-instrumentations-node");
const { OTLPTraceExporter } = require("@opentelemetry/exporter-trace-otlp-http");

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: "http://otel-collector:4318/v1/traces"
  }),
  instrumentations: [getNodeAutoInstrumentations()]
});

sdk.start();
```

Run your application with:

```bash
node -r ./tracing.js app.js
```

This automatically creates spans for every MongoDB operation with attributes like `db.system: mongodb`, `db.name`, and `db.operation`.

## Instrumenting Python Applications

```bash
pip install opentelemetry-instrumentation-pymongo
```

```python
from opentelemetry.instrumentation.pymongo import PymongoInstrumentor

PymongoInstrumentor().instrument()

# All subsequent pymongo calls are traced automatically
from pymongo import MongoClient
client = MongoClient("mongodb://localhost:27017")
db = client["mydb"]
db.orders.find_one({"status": "completed"})
```

## Viewing MongoDB Traces

Each traced MongoDB operation appears as a span with:
- `db.system: mongodb`
- `db.name: mydb`
- `db.operation: find`
- `db.statement: {"status": "completed"}` (if statement capture is enabled)

## Summary

Monitor MongoDB with OpenTelemetry by using the Collector's MongoDB receiver for server-level metrics and auto-instrumentation for application-level traces. This gives you full observability from the database server stats down to individual query spans in your distributed traces. Send the data to any OTLP-compatible backend like Grafana, Jaeger, or OneUptime.
