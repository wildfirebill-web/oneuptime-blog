# How to Set Up MongoDB Monitoring on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Kubernetes, Monitoring, Prometheus, Metric

Description: Set up comprehensive MongoDB monitoring on Kubernetes using the MongoDB Exporter, Prometheus, and ServiceMonitor resources for full observability.

---

## Introduction

Running MongoDB on Kubernetes without proper monitoring leaves you blind to performance degradation, connection exhaustion, and replication lag. This guide covers deploying the MongoDB Exporter alongside your MongoDB StatefulSet and scraping metrics with Prometheus.

## Prerequisites

- A running MongoDB StatefulSet on Kubernetes
- Prometheus Operator installed in the cluster
- `kubectl` access with appropriate permissions

## Deploying the MongoDB Exporter

The `mongodb_exporter` by Percona exposes MongoDB metrics in Prometheus format. Deploy it as a sidecar or standalone Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-exporter
  namespace: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb-exporter
  template:
    metadata:
      labels:
        app: mongodb-exporter
    spec:
      containers:
        - name: mongodb-exporter
          image: percona/mongodb_exporter:0.40
          args:
            - --mongodb.uri=$(MONGODB_URI)
            - --collect-all
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: uri
          ports:
            - containerPort: 9216
              name: metrics
```

## Creating the Metrics Secret

```bash
kubectl create secret generic mongodb-credentials \
  --from-literal=uri="mongodb://monitor:monitorPass@mongodb-0.mongodb:27017/admin" \
  -n mongodb
```

## Exposing Metrics via a Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-exporter
  namespace: mongodb
  labels:
    app: mongodb-exporter
spec:
  ports:
    - name: metrics
      port: 9216
      targetPort: 9216
  selector:
    app: mongodb-exporter
```

## Creating a ServiceMonitor

If you use the Prometheus Operator, create a `ServiceMonitor` to auto-discover the exporter:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mongodb-exporter
  namespace: mongodb
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: mongodb-exporter
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

## Creating a MongoDB Monitor User

```javascript
use admin
db.createUser({
  user: "monitor",
  pwd: "monitorPass",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }
  ]
})
```

## Key Metrics to Watch

Once scraping is active, focus on these metrics:

```text
mongodb_up                          - Exporter connectivity
mongodb_connections{state="current"} - Active connections
mongodb_opcounters_total            - Operations per second
mongodb_repl_lag_seconds            - Replica set lag
mongodb_mem_resident_mb             - Resident memory usage
mongodb_wiredtiger_cache_used_bytes - WiredTiger cache pressure
```

## Verifying the Setup

```bash
# Port-forward to exporter
kubectl port-forward svc/mongodb-exporter 9216:9216 -n mongodb

# Check metrics
curl http://localhost:9216/metrics | grep mongodb_up
```

## Summary

Setting up MongoDB monitoring on Kubernetes involves deploying the MongoDB Exporter, creating a dedicated monitoring user with minimal privileges, exposing metrics via a Service, and configuring a ServiceMonitor for Prometheus scraping. This foundation enables you to build dashboards in Grafana, configure alerts on replication lag, and track connection pool exhaustion before it affects your application.
