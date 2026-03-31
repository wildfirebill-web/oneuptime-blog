# How to Set Up Ceph Storage for Grafana Loki on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Grafana, Loki, Kubernetes, Storage, Logging

Description: Configure Grafana Loki to use Ceph object storage via Rook RGW on Kubernetes for scalable, cost-efficient log storage with S3-compatible chunks backend.

---

## Why Ceph for Loki

Grafana Loki supports multiple storage backends for chunks and indexes. Ceph RGW's S3-compatible interface makes it a natural fit for Loki's object store backend, allowing you to keep log data on-premises while using the same S3 API Loki already supports.

## Set Up Ceph Object Store

First, create a CephObjectStore and expose it via a service:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: loki-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  preservePoolsOnDelete: false
  gateway:
    port: 80
    instances: 2
```

Create an object store user for Loki:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
  --uid=loki \
  --display-name="Loki User" \
  --access-key=lokiaccesskey \
  --secret-key=lokisecretkey
```

Create buckets for chunks and ruler:

```bash
aws s3 mb s3://loki-chunks --endpoint-url http://rook-ceph-rgw-loki-store.rook-ceph:80
aws s3 mb s3://loki-ruler  --endpoint-url http://rook-ceph-rgw-loki-store.rook-ceph:80
```

## Deploy Loki with Ceph S3 Backend

Install Loki using Helm with S3 config:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Create a values file `loki-values.yaml`:

```yaml
loki:
  auth_enabled: false
  storage:
    type: s3
    s3:
      endpoint: http://rook-ceph-rgw-loki-store.rook-ceph:80
      region: us-east-1
      bucketnames: loki-chunks
      access_key_id: lokiaccesskey
      secret_access_key: lokisecretkey
      s3forcepathstyle: true
      insecure: true
  schema_config:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: index_
          period: 24h
```

```bash
helm install loki grafana/loki \
  --namespace logging \
  --create-namespace \
  -f loki-values.yaml
```

## Verify Log Ingestion

Forward the Loki port and push a test log:

```bash
kubectl -n logging port-forward svc/loki 3100
curl -X POST http://localhost:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{"streams":[{"stream":{"app":"test"},"values":[["'$(date +%s%N)'","hello from ceph"]]}]}'
```

Query logs back:

```bash
curl "http://localhost:3100/loki/api/v1/query_range?query={app='test'}&limit=5" | jq '.data.result'
```

## Summary

Using Ceph RGW as Loki's S3-compatible object store keeps log data on-premises while leveraging Loki's proven chunked storage design. This setup scales horizontally by adding more Ceph OSDs and RGW instances, making it an excellent choice for large Kubernetes logging pipelines.
