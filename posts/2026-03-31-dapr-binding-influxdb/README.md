# How to Use Dapr InfluxDB Output Binding for Time-Series Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Binding, InfluxDB, Time-Series, Observability

Description: Learn how to configure the Dapr InfluxDB output binding to write time-series metrics and events from microservices to an InfluxDB instance.

---

## Why InfluxDB for Time-Series Data

InfluxDB is optimized for high-frequency time-series data like metrics, IoT sensor readings, and application performance data. The Dapr InfluxDB binding lets any service write data points without managing InfluxDB client libraries.

## Start InfluxDB Locally

```bash
docker run -d \
  --name influxdb \
  -p 8086:8086 \
  -e INFLUXDB_DB=metrics \
  -e INFLUXDB_ADMIN_USER=admin \
  -e INFLUXDB_ADMIN_PASSWORD=adminpass \
  influxdb:1.8
```

## Configure the InfluxDB Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: influxdb-metrics
spec:
  type: bindings.influx
  version: v1
  metadata:
  - name: url
    value: http://localhost:8086
  - name: token
    secretKeyRef:
      name: influxdb-secret
      key: token
  - name: org
    value: myorg
  - name: bucket
    value: metrics
```

For InfluxDB 1.x, use the legacy auth format:

```yaml
metadata:
- name: url
  value: http://localhost:8086
- name: dbName
  value: metrics
- name: username
  value: admin
- name: password
  secretKeyRef:
    name: influxdb-secret
    key: password
```

## Write a Data Point

```bash
curl -X POST http://localhost:3500/v1.0/bindings/influxdb-metrics \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "create",
    "data": "cpu_usage,host=server-01,region=us-east value=72.5 1711875600000000000"
  }'
```

The data field follows InfluxDB Line Protocol: `measurement,tag=value field=value timestamp`.

## Write Multiple Data Points

```bash
curl -X POST http://localhost:3500/v1.0/bindings/influxdb-metrics \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "create",
    "data": "cpu,host=web-01 value=45.2\nmemory,host=web-01 used=2048,total=8192\ndisk,host=web-01,path=/ used_percent=68.4"
  }'
```

## Application Code for Writing Metrics

```python
import time
import requests

class MetricsWriter:
    def __init__(self, dapr_port: int = 3500):
        self.url = f"http://localhost:{dapr_port}/v1.0/bindings/influxdb-metrics"

    def write(self, measurement: str, fields: dict, tags: dict = None):
        tags_str = ""
        if tags:
            tags_str = "," + ",".join(f"{k}={v}" for k, v in tags.items())

        fields_str = ",".join(f"{k}={v}" for k, v in fields.items())
        timestamp_ns = int(time.time() * 1e9)
        line = f"{measurement}{tags_str} {fields_str} {timestamp_ns}"

        requests.post(
            self.url,
            json={"operation": "create", "data": line},
        )

metrics = MetricsWriter()

# Write application metrics
metrics.write(
    measurement="request_duration",
    fields={"duration_ms": 142, "status": 200},
    tags={"service": "order-api", "endpoint": "/orders"},
)
```

## Writing Batches Efficiently

```python
def write_batch(measurements: list[str]):
    line_protocol = "\n".join(measurements)
    requests.post(
        "http://localhost:3500/v1.0/bindings/influxdb-metrics",
        json={"operation": "create", "data": line_protocol},
    )

# Collect and flush in batches
buffer = []
for reading in sensor_readings:
    ts = int(reading.timestamp * 1e9)
    buffer.append(f"temperature,sensor={reading.id} value={reading.temp} {ts}")
    if len(buffer) >= 100:
        write_batch(buffer)
        buffer = []
```

## Summary

The Dapr InfluxDB output binding enables any microservice to write time-series data using InfluxDB Line Protocol. Configure the URL, authentication, and bucket in the component YAML, then POST line-protocol data strings to the binding endpoint. Batch multiple points in a single call for efficient high-frequency writes.
