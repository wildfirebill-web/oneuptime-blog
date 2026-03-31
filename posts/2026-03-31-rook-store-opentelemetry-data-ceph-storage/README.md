# How to Store OpenTelemetry Data on Ceph Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OpenTelemetry, Kubernetes, Storage, Observability, OTEL

Description: Use Ceph RGW and RBD storage on Kubernetes to persist OpenTelemetry collector data, including traces, metrics, and logs exported to S3-compatible backends.

---

## Overview

The OpenTelemetry Collector acts as a pipeline for telemetry data. It supports exporting to object storage via the `otlpexporter` and file exporters. Ceph provides both block (RBD) and object (RGW) storage to persist OTEL data on Kubernetes.

## Option 1 - Export Traces to Ceph via File Exporter

The `fileexporter` writes OTLP JSON to disk. Mount a Ceph RBD PVC to the collector pod:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: otel-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-rbd
  resources:
    requests:
      storage: 10Gi
```

Configure the collector to write to the mounted path:

```yaml
exporters:
  file:
    path: /otel-data/traces.json
    rotation:
      max_megabytes: 100
      max_days: 7

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [file]
```

## Option 2 - Export to S3 via Ceph RGW

Use the `awss3exporter` (community extension) to ship data directly to Ceph RGW:

Install the contrib distribution of the OTEL Collector:

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace otel \
  --create-namespace \
  --set image.repository=otel/opentelemetry-collector-contrib
```

Configure `otelcol-config.yaml`:

```yaml
exporters:
  awss3:
    s3uploader:
      region: us-east-1
      s3_bucket: otel-traces
      s3_prefix: traces
      endpoint: http://rook-ceph-rgw-otel-store.rook-ceph:80
      s3_force_path_style: true
    marshaler: otlp_json

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [awss3]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [awss3]
```

## Set Up Ceph Bucket for OTEL Data

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
  --uid=otelcollector \
  --display-name="OTEL Collector" \
  --access-key=otelaccesskey \
  --secret-key=otelsecretkey

aws s3 mb s3://otel-traces \
  --endpoint-url http://rook-ceph-rgw-otel-store.rook-ceph:80
```

## Verify Data in Ceph

```bash
aws s3 ls s3://otel-traces/ \
  --endpoint-url http://rook-ceph-rgw-otel-store.rook-ceph:80 \
  --recursive | head -10
```

Inspect a stored trace file:

```bash
aws s3 cp s3://otel-traces/traces/2026-03-31/trace-0001.json - \
  --endpoint-url http://rook-ceph-rgw-otel-store.rook-ceph:80 | jq .
```

## Summary

Ceph supports OpenTelemetry data storage via both block (RBD for file exporter) and object (RGW for S3 exporter) interfaces. This flexibility lets you choose the right storage tier for each telemetry signal, all managed declaratively through Rook on Kubernetes.
