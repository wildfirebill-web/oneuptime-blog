# How to Monitor MongoDB with Prometheus and Grafana

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Prometheus, Grafana, Monitoring, Metrics

Description: Learn how to export MongoDB metrics to Prometheus using mongodb_exporter and visualize them with Grafana dashboards for connection, operation, and replication monitoring.

---

## Overview

Monitoring MongoDB with Prometheus and Grafana gives you visibility into connection pool usage, operation latency, replication lag, and storage metrics. The `mongodb_exporter` by Percona scrapes MongoDB's internal stats and exposes them in Prometheus format.

## Installing mongodb_exporter

Using Docker:

```bash
docker run -d \
  --name mongodb-exporter \
  -p 9216:9216 \
  -e MONGODB_URI="mongodb://admin:password@localhost:27017" \
  percona/mongodb_exporter:0.40
```

Or install directly:

```bash
wget https://github.com/percona/mongodb_exporter/releases/latest/download/mongodb_exporter-linux-amd64.tar.gz
tar -xzf mongodb_exporter-linux-amd64.tar.gz
./mongodb_exporter --mongodb.uri="mongodb://admin:password@localhost:27017" --web.listen-address=":9216"
```

## Deploying on Kubernetes with Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter \
  --namespace monitoring \
  --set mongodb.uri="mongodb://admin:password@mongodb-svc.mongodb.svc.cluster.local:27017"
```

## Configuring Prometheus to Scrape MongoDB Metrics

Add a scrape config to `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "mongodb"
    static_configs:
      - targets: ["mongodb-exporter:9216"]
    scrape_interval: 30s
    scrape_timeout: 10s
```

## Key Metrics to Monitor

Important metrics exposed by `mongodb_exporter`:

```
# Connection pool
mongodb_connections{state="current"}
mongodb_connections{state="available"}

# Operations per second
mongodb_op_counters_total{type="insert"}
mongodb_op_counters_total{type="query"}
mongodb_op_counters_total{type="update"}
mongodb_op_counters_total{type="delete"}

# Replication lag
mongodb_replication_lag_seconds

# Memory usage
mongodb_memory{type="resident"}
mongodb_memory{type="virtual"}

# Page faults
mongodb_extra_info_page_faults_total
```

## Setting Up Grafana Alerts

Example Prometheus alert rules:

```yaml
groups:
  - name: mongodb
    rules:
      - alert: MongoDBReplicationLagHigh
        expr: mongodb_replication_lag_seconds > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB replication lag is above 30 seconds"

      - alert: MongoDBConnectionsNearMax
        expr: >
          mongodb_connections{state="current"} /
          (mongodb_connections{state="current"} + mongodb_connections{state="available"})
          > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB connection pool usage above 85%"
```

## Importing a Grafana Dashboard

Use the community MongoDB dashboard (ID 2583 or 7353) from Grafana's dashboard library:

1. Go to Dashboards - Import
2. Enter ID `7353` and click Load
3. Select your Prometheus data source
4. Click Import

## Summary

Monitor MongoDB with Prometheus by deploying `mongodb_exporter` to scrape server stats and exposing a Prometheus scrape endpoint. Configure alert rules for replication lag and connection pool saturation. Import a Grafana community dashboard to get pre-built visualizations for connections, operations, memory, and replication health within minutes.
