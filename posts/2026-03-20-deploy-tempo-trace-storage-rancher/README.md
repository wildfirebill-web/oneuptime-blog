# How to Deploy Tempo on Rancher for Trace Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Grafana Tempo, Distributed Tracing, Observability, Helm, Object Storage

Description: Deploy Grafana Tempo on Rancher for cost-effective trace storage with object storage backend and Grafana integration for trace visualization.

## Introduction

Grafana Tempo is a high-volume distributed tracing backend that requires only object storage (no separate index). Unlike Jaeger with Elasticsearch, Tempo stores traces directly in object storage like S3, making it significantly cheaper to operate at scale.

## Prerequisites

- Rancher cluster with `helm` and `kubectl`
- S3-compatible object storage
- Grafana installed (for visualization)

## Step 1: Add Grafana Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## Step 2: Create Object Storage Secret

```bash
kubectl create secret generic tempo-s3-credentials \
  --from-literal=access_key_id=YOUR_ACCESS_KEY \
  --from-literal=secret_access_key=YOUR_SECRET_KEY \
  -n observability
```

## Step 3: Configure Tempo Values

```yaml
# tempo-values.yaml

tempo:
  storage:
    trace:
      backend: s3
      s3:
        bucket: my-tempo-traces
        endpoint: s3.amazonaws.com
        region: us-east-1
        access_key: ${S3_ACCESS_KEY}
        secret_key: ${S3_SECRET_KEY}

  ingester:
    replicas: 3
    trace_idle_period: 30s
    max_block_bytes: 1073741824   # 1GB blocks

  distributor:
    replicas: 2
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

  compactor:
    compaction:
      block_retention: 168h    # Keep traces for 7 days

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "2Gi"
    cpu: "1"
```

## Step 4: Deploy Tempo

```bash
helm install tempo grafana/tempo-distributed \
  --namespace observability \
  --values tempo-values.yaml
```

## Step 5: Verify Deployment

```bash
# Check all Tempo components
kubectl get pods -n observability | grep tempo

# Verify ingester is ready
kubectl logs -n observability tempo-ingester-0 | grep "starting ingester"
```

## Step 6: Configure Grafana Data Source

Add Tempo as a data source in Grafana for trace visualization:

```yaml
# Grafana data source ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: observability
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Tempo
        type: tempo
        url: http://tempo-query-frontend.observability.svc.cluster.local:3100
        jsonData:
          httpMethod: GET
          tracesToLogsV2:
            datasourceUid: loki    # Link traces to Loki logs
            tags: ['service.name', 'pod']
```

## Step 7: Send Traces via OpenTelemetry

```yaml
# Configure OTel Collector to forward to Tempo
exporters:
  otlp/tempo:
    endpoint: tempo-distributor.observability.svc.cluster.local:4317
    tls:
      insecure: true
```

## Conclusion

Grafana Tempo on Rancher provides cost-effective trace storage by eliminating the need for Elasticsearch or Cassandra. Object storage costs are a fraction of block storage, making Tempo economical even at millions of traces per day. The tight Grafana integration enables seamless navigation from metrics to traces to logs.
