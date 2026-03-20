# How to Deploy Tempo on Rancher for Trace Storage - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Tempo, Distributed Tracing, Grafana, Observability

Description: Deploy Grafana Tempo on Rancher for cost-effective, scalable distributed trace storage that integrates natively with Grafana for trace visualization.

## Introduction

Grafana Tempo is a distributed tracing backend that stores traces in object storage (S3, GCS, Azure Blob) at minimal cost. Unlike Jaeger or Zipkin, Tempo doesn't index traces for ad-hoc search-instead, it uses trace IDs from logs and metrics to look up specific traces. This guide covers deploying Tempo on Rancher with the Grafana LGTM stack.

## Prerequisites

- Rancher-managed Kubernetes cluster
- Helm 3.x installed
- Object storage (S3 or MinIO)
- Grafana (from Rancher Monitoring)

## Step 1: Deploy MinIO for Local Object Storage

For on-premises deployments without cloud object storage:

```bash
# Deploy MinIO for Tempo backend

helm repo add minio https://charts.min.io/
helm install minio minio/minio \
  --namespace observability \
  --create-namespace \
  --set rootUser=minioadmin \
  --set rootPassword=minioadmin \
  --set persistence.size=100Gi \
  --set mode=standalone \
  --wait

# Create the tempo bucket
kubectl exec -n observability deployment/minio -- \
  mc alias set local http://localhost:9000 minioadmin minioadmin

kubectl exec -n observability deployment/minio -- \
  mc mb local/tempo
```

## Step 2: Deploy Tempo with Helm

```yaml
# tempo-values.yaml - Tempo distributed configuration
tempo:
  # Storage configuration
  storage:
    trace:
      backend: s3
      s3:
        bucket: tempo
        endpoint: minio.observability.svc.cluster.local:9000
        access_key: minioadmin
        secret_key: minioadmin
        insecure: true

  # Ingester configuration
  ingester:
    lifecycler:
      ring:
        replication_factor: 2

  # Compactor runs hourly
  compactor:
    compaction:
      block_retention: 720h  # 30 days

  # Server configuration
  server:
    http_listen_port: 3100
    grpc_listen_port: 9095

  # Distributor (receives traces)
  distributor:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      jaeger:
        protocols:
          thrift_http:
            endpoint: 0.0.0.0:14268
          grpc:
            endpoint: 0.0.0.0:14250
      zipkin:
        endpoint: 0.0.0.0:9411

  # Query configuration
  querier:
    max_concurrent_queries: 20

  # Metrics generation (connect traces to metrics)
  metricsGenerator:
    enabled: true
    storage:
      path: /var/tempo/wal
      remote_write:
        - url: http://rancher-monitoring-prometheus.cattle-monitoring-system.svc.cluster.local:9090/api/v1/write
          send_exemplars: true

serviceMonitor:
  enabled: true
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
```

```bash
# Add Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Tempo
helm install tempo grafana/tempo-distributed \
  --namespace observability \
  --create-namespace \
  --values tempo-values.yaml \
  --wait

# Check pods
kubectl get pods -n observability -l app.kubernetes.io/name=tempo
```

## Step 3: Configure Grafana to Use Tempo

```yaml
# grafana-tempo-datasource.yaml - Tempo data source for Grafana
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-tempo
  namespace: cattle-monitoring-system
  labels:
    grafana_datasource: "1"
data:
  tempo-datasource.yaml: |
    apiVersion: 1
    datasources:
      - name: Tempo
        type: tempo
        url: http://tempo-query-frontend.observability.svc.cluster.local:3100
        access: proxy
        jsonData:
          tracesToLogsV2:
            datasourceUid: loki
            spanStartTimeShift: -1h
            spanEndTimeShift: 1h
            filterByTraceID: true
            filterBySpanID: false
            customQuery: true
            query: '{namespace="${__span.tags.k8s.namespace.name}"}'
          tracesToMetrics:
            datasourceUid: prometheus
            spanStartTimeShift: -1h
            spanEndTimeShift: 1h
          serviceMap:
            datasourceUid: prometheus
          nodeGraph:
            enabled: true
          search:
            hide: false
          lokiSearch:
            datasourceUid: loki
```

## Step 4: Configure OpenTelemetry Collector to Send to Tempo

```yaml
# otel-tempo-config.yaml - OTel Collector routing to Tempo
config:
  exporters:
    otlp/tempo:
      endpoint: tempo-distributor.observability.svc.cluster.local:4317
      tls:
        insecure: true

  service:
    pipelines:
      traces:
        receivers: [otlp, jaeger]
        processors: [memory_limiter, k8sattributes, batch]
        exporters: [otlp/tempo]  # Send to Tempo
```

## Step 5: Enable Trace to Logs Correlation

```yaml
# loki-with-trace-id.yaml - Loki config to include trace IDs in logs
config:
  distributor:
    ring:
      kvstore:
        store: memberlist
  schema_config:
    configs:
      - from: 2024-01-01
        store: boltdb-shipper
        object_store: s3
        schema: v12
        index:
          prefix: index_
          period: 24h
```

Configure your applications to include trace IDs in log output:

```python
# Python logging with trace correlation
import logging
from opentelemetry import trace

class TraceIdInjectionFilter(logging.Filter):
    def filter(self, record):
        span = trace.get_current_span()
        if span.is_recording():
            ctx = span.get_span_context()
            record.trace_id = format(ctx.trace_id, '032x')
            record.span_id = format(ctx.span_id, '016x')
        else:
            record.trace_id = '0' * 32
            record.span_id = '0' * 16
        return True

# Configure JSON formatter with trace ID
logging.basicConfig(
    format='{"timestamp":"%(asctime)s","level":"%(levelname)s","msg":"%(message)s","trace_id":"%(trace_id)s","span_id":"%(span_id)s"}'
)
logging.getLogger().addFilter(TraceIdInjectionFilter())
```

## Step 6: Test the Trace Pipeline

```bash
# Send a test trace
kubectl exec -n observability deployment/otel-collector -- \
  curl -s -X POST \
  http://tempo-distributor.observability.svc.cluster.local:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "test-service"}}]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "7bba9f33312b3dbb8b2c2c62bb7abe2d",
          "spanId": "086e83747d0e381e",
          "name": "test-span",
          "startTimeUnixNano": "1609459200000000000",
          "endTimeUnixNano": "1609459200100000000",
          "status": {}
        }]
      }]
    }]
  }'

# Search for the trace in Tempo
kubectl port-forward -n observability svc/tempo-query-frontend 3100:3100 &
curl "http://localhost:3100/api/traces/7bba9f33312b3dbb8b2c2c62bb7abe2d" | jq '.'
```

## Conclusion

Grafana Tempo provides cost-effective distributed trace storage that excels when used alongside Grafana, Loki, and Prometheus in the LGTM (Loki, Grafana, Tempo, Mimir) stack. Its object-storage backend keeps costs low even for high trace volumes. The exemplar-based correlation between metrics, logs, and traces provides a seamless troubleshooting experience in Grafana, allowing you to jump from a slow metric to the specific trace and log entries that explain the slowdown.
