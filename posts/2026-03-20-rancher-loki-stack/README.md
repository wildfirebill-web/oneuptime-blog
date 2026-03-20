# How to Deploy Loki Stack on Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Loki, Log Aggregation, Grafana, Observability

Description: Deploy the Grafana Loki stack on Rancher for scalable, cost-effective log aggregation with Promtail for log collection and Grafana for visualization.

## Introduction

Grafana Loki is a horizontally scalable log aggregation system inspired by Prometheus. Unlike ELK, Loki only indexes metadata (labels) rather than full log content, making it significantly cheaper to operate at scale. This guide covers deploying the complete Loki stack (Loki + Promtail + Grafana) on Rancher.

## Prerequisites

- Rancher-managed Kubernetes cluster
- Helm 3.x installed
- kubectl access
- A StorageClass or object storage backend

## Step 1: Deploy Loki with Helm

```yaml
# loki-values.yaml - Production Loki configuration

loki:
  auth_enabled: false  # Set to true for multi-tenancy

  commonConfig:
    replication_factor: 2

  storage:
    type: s3
    s3:
      endpoint: minio.observability.svc.cluster.local:9000
      bucketnames: loki-chunks
      access_key_id: minioadmin
      secret_access_key: minioadmin
      s3forcepathstyle: true
      insecure: true

  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  limits_config:
    # Retention period
    retention_period: 30d
    # Ingestion rate limit
    ingestion_rate_mb: 16
    ingestion_burst_size_mb: 32
    # Query limits
    max_query_series: 5000
    max_entries_limit_per_query: 50000

  # Compaction
  compactor:
    retention_enabled: true
    retention_delete_delay: 2h
    compaction_interval: 10m

  # Ruler for log-based alerting
  ruler:
    alertmanager_url: http://rancher-monitoring-alertmanager.cattle-monitoring-system.svc.cluster.local:9093

singleBinary:
  replicas: 3
  persistence:
    enabled: true
    size: 10Gi

serviceMonitor:
  enabled: true
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
```

```bash
# Install Loki
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki \
  --namespace observability \
  --create-namespace \
  --values loki-values.yaml \
  --wait

# Check status
kubectl get pods -n observability -l app.kubernetes.io/name=loki
```

## Step 2: Deploy Promtail for Log Collection

```yaml
# promtail-values.yaml - Promtail DaemonSet configuration
config:
  # Send logs to Loki
  clients:
    - url: http://loki.observability.svc.cluster.local:3100/loki/api/v1/push

  # Log scrape configuration
  scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
        - role: pod
      relabel_configs:
        # Include only pods with annotation
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: drop
          regex: "false"
        # Add pod labels as Loki labels
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod
        - source_labels: [__meta_kubernetes_container_name]
          target_label: container
        - source_labels: [__meta_kubernetes_pod_label_app]
          target_label: app
        # Add node name
        - source_labels: [__meta_kubernetes_pod_node_name]
          target_label: node
      pipeline_stages:
        # Parse JSON logs
        - json:
            expressions:
              level: level
              timestamp: timestamp
              msg: msg
              trace_id: trace_id
        - labels:
            level:
        - timestamp:
            source: timestamp
            format: RFC3339
        # Drop debug logs in production
        - drop:
            expression: '.*level=debug.*'

    # System logs
    - job_name: system-logs
      static_configs:
        - targets:
            - localhost
          labels:
            job: varlogs
            __path__: /var/log/*log
```

```bash
# Install Promtail
helm install promtail grafana/promtail \
  --namespace observability \
  --values promtail-values.yaml \
  --wait
```

## Step 3: Configure Grafana Data Source

```yaml
# grafana-loki-datasource.yaml - Loki data source for Grafana
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-loki
  namespace: cattle-monitoring-system
  labels:
    grafana_datasource: "1"
data:
  loki-datasource.yaml: |
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        url: http://loki.observability.svc.cluster.local:3100
        access: proxy
        jsonData:
          maxLines: 1000
          derivedFields:
            # Link trace_id in logs to Tempo traces
            - datasourceUid: tempo
              matcherRegex: '"trace_id":"(\w+)"'
              name: TraceID
              url: '$${__value.raw}'
```

## Step 4: Write LogQL Queries

LogQL is Loki's query language, similar to PromQL:

```bash
# Query all error logs from production namespace
{namespace="production"} |= "error"

# Filter by multiple labels and level
{namespace="production", app="order-service"} |= "ERROR"

# Parse JSON and filter on structured field
{namespace="production"} | json | level = "error" | line_format "{{.msg}}"

# Count errors per service (metric query)
sum by (app) (
  rate({namespace="production"} |= "ERROR" [5m])
)

# High latency requests (parsing nginx access logs)
{job="nginx-access"}
  | logfmt
  | response_time > 2.0
  | line_format "{{.method}} {{.path}} took {{.response_time}}s"
```

## Step 5: Create Log-Based Alerts

```yaml
# loki-alerts.yaml - Alert rules using LogQL
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: log-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: log-based-alerts
      rules:
        # Alert on high error rate in logs
        - alert: HighErrorRateInLogs
          expr: |
            sum by (app, namespace) (
              rate({namespace="production"} |= "ERROR" [5m])
            ) > 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High error rate in {{ $labels.app }} logs"
```

## Step 6: Configure Log Retention

```bash
# Set retention via Loki admin API
curl -X POST http://loki.observability.svc.cluster.local:3100/loki/api/v1/delete \
  -H "Content-Type: application/json" \
  -d '{
    "match[]": "{namespace=\"development\"}",
    "start": "2026-01-01T00:00:00Z",
    "end": "2026-01-31T23:59:59Z"
  }'
```

## Troubleshooting

```bash
# Check Loki health
curl http://loki.observability.svc.cluster.local:3100/ready

# Check Promtail targets
kubectl port-forward -n observability \
  daemonset/promtail 3101:3101 &
curl http://localhost:3101/targets

# Check ingestion rate
curl http://loki.observability.svc.cluster.local:3100/metrics | \
  grep loki_ingester_streams_created_total
```

## Conclusion

The Loki stack provides a cost-effective, Kubernetes-native log aggregation solution that integrates seamlessly with Rancher's monitoring infrastructure. Its label-based indexing makes it dramatically cheaper than Elasticsearch-based solutions for log storage at scale. When combined with Tempo for traces and Prometheus/Mimir for metrics, Loki completes a full observability stack that enables correlation between logs, metrics, and traces in Grafana.
