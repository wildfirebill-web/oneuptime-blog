# How to Deploy Mimir on Rancher for Metrics Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Grafana Mimir, Prometheus, Long-Term Metrics, Observability, Helm

Description: Deploy Grafana Mimir on Rancher for scalable, long-term Prometheus-compatible metrics storage using object storage as the backend.

## Introduction

Grafana Mimir is a horizontally scalable, highly available metrics storage system compatible with the Prometheus remote write API. It scales to billions of metric series and provides unlimited long-term retention using object storage—making it the modern replacement for Thanos in new deployments.

## Prerequisites

- Rancher cluster with sufficient resources (Mimir has many components)
- S3-compatible object storage
- Prometheus configured for remote write

## Step 1: Add Grafana Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## Step 2: Create Storage Secret

```bash
kubectl create secret generic mimir-bucket-secret \
  --from-literal=AWS_ACCESS_KEY_ID=YOUR_KEY \
  --from-literal=AWS_SECRET_ACCESS_KEY=YOUR_SECRET \
  -n monitoring
```

## Step 3: Configure Mimir Values

```yaml
# mimir-values.yaml
mimir:
  structuredConfig:
    common:
      storage:
        backend: s3
        s3:
          bucket_name: mimir-metrics
          endpoint: s3.amazonaws.com
          region: us-east-1
          access_key_id: ${AWS_ACCESS_KEY_ID}
          secret_access_key: ${AWS_SECRET_ACCESS_KEY}

    blocks_storage:
      s3:
        bucket_name: mimir-blocks

    alertmanager_storage:
      s3:
        bucket_name: mimir-alertmanager

    ruler_storage:
      s3:
        bucket_name: mimir-ruler

# Component scaling
distributor:
  replicas: 2
ingester:
  replicas: 3
  zoneAwareReplication:
    enabled: false
querier:
  replicas: 2
query_frontend:
  replicas: 2
compactor:
  replicas: 1
store_gateway:
  replicas: 2

# Minimal mode for smaller clusters
minio:
  enabled: false    # Use external S3, not embedded MinIO
```

## Step 4: Deploy Mimir

```bash
kubectl create namespace monitoring

helm install mimir grafana/mimir-distributed \
  --namespace monitoring \
  --values mimir-values.yaml \
  --timeout 10m
```

## Step 5: Configure Prometheus Remote Write

Point Prometheus to send metrics to Mimir:

```yaml
# prometheus-values.yaml (remote_write section)
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
        headers:
          X-Scope-OrgID: myorg    # Multi-tenancy header
        queueConfig:
          maxSamplesPerSend: 10000
          capacity: 50000
```

## Step 6: Query Metrics via Grafana

Configure Grafana to use Mimir as a Prometheus data source:

```yaml
datasources:
  - name: Mimir
    type: prometheus
    url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
    jsonData:
      httpHeaderName1: X-Scope-OrgID
    secureJsonData:
      httpHeaderValue1: myorg
```

## Conclusion

Grafana Mimir on Rancher provides Prometheus-compatible metrics storage that scales horizontally across all its components. Object storage backend keeps costs manageable, while native multi-tenancy support makes it suitable for shared platform teams serving multiple application teams.
