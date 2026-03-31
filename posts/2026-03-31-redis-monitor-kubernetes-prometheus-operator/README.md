# How to Monitor Redis in Kubernetes with Prometheus Operator

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Kubernetes, Prometheus, Monitoring, ServiceMonitor

Description: Set up Redis monitoring in Kubernetes using the Prometheus Operator, redis_exporter, and ServiceMonitor to collect metrics and trigger alerts automatically.

---

The Prometheus Operator extends Kubernetes with custom resources like `ServiceMonitor` and `PrometheusRule`, making it easy to configure scraping and alerting without editing Prometheus config files directly. This guide walks through deploying the Redis exporter and connecting it to the Prometheus Operator.

## Prerequisites

- Prometheus Operator installed (via kube-prometheus-stack or standalone)
- Redis running in Kubernetes
- `kubectl` access with appropriate permissions

## Step 1: Deploy the Redis Exporter

Deploy `redis_exporter` as a sidecar or standalone deployment. Standalone is easier to manage:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
  namespace: redis
  labels:
    app: redis-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-exporter
  template:
    metadata:
      labels:
        app: redis-exporter
    spec:
      containers:
      - name: redis-exporter
        image: oliver006/redis_exporter:v1.62.0
        args:
        - --redis.addr=redis://redis-master.redis.svc.cluster.local:6379
        - --redis.password=$(REDIS_PASSWORD)
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
        ports:
        - containerPort: 9121
          name: metrics
```

## Step 2: Expose the Exporter as a Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-exporter
  namespace: redis
  labels:
    app: redis-exporter
spec:
  selector:
    app: redis-exporter
  ports:
  - name: metrics
    port: 9121
    targetPort: 9121
```

## Step 3: Create a ServiceMonitor

The `ServiceMonitor` tells the Prometheus Operator which services to scrape:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-monitor
  namespace: redis
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: redis-exporter
  endpoints:
  - port: metrics
    interval: 30s
    scrapeTimeout: 10s
    path: /metrics
```

Apply all resources:

```bash
kubectl apply -f redis-exporter-deployment.yaml
kubectl apply -f redis-exporter-service.yaml
kubectl apply -f redis-servicemonitor.yaml
```

## Step 4: Verify Scraping

Check that Prometheus has discovered the target:

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring
```

Then open `http://localhost:9090/targets` and look for `redis-monitor`. You can also query metrics:

```text
redis_up
redis_connected_clients
redis_memory_used_bytes
redis_commands_total
```

## Step 5: Create a PrometheusRule for Alerting

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: redis-alerts
  namespace: redis
  labels:
    release: prometheus
spec:
  groups:
  - name: redis.rules
    rules:
    - alert: RedisDown
      expr: redis_up == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Redis instance is down"
        description: "Redis {{ $labels.instance }} has been down for more than 1 minute."
    - alert: RedisHighMemoryUsage
      expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Redis memory usage above 90%"
        description: "Redis {{ $labels.instance }} is using {{ $value | humanizePercentage }} of its max memory."
```

## Step 6: Import a Grafana Dashboard

Use the official Redis dashboard (ID 763) from Grafana.com:

```bash
# In Grafana UI: Dashboards -> Import -> Enter ID 763
# Or via Grafana Operator GrafanaDashboard CRD
```

## Summary

Monitoring Redis with the Prometheus Operator involves three components: the `redis_exporter` to expose metrics, a `ServiceMonitor` to configure scraping, and `PrometheusRule` for alerting. This declarative approach lets you manage all monitoring configuration as Kubernetes manifests alongside your application code.
