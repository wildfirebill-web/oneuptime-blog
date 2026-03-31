# How to Set Up Ceph Object Storage for Thanos on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Thanos, Kubernetes, Storage, Prometheus, Object Storage

Description: Use Ceph RGW as the S3-compatible object store for Thanos on Kubernetes to achieve long-term Prometheus metrics retention without relying on cloud object storage.

---

## Overview

Thanos extends Prometheus with long-term storage by uploading TSDB blocks to object storage. Ceph RGW's S3 API compatibility lets you configure Thanos to use an on-premises Ceph cluster as its object store, delivering cloud-like scalability on your own infrastructure.

## Prepare the Ceph Object Store

Deploy a CephObjectStore via Rook:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: thanos-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    erasureCoded:
      dataChunks: 4
      codingChunks: 2
  gateway:
    port: 80
    instances: 3
```

Create object store user and Thanos bucket:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
  --uid=thanos \
  --display-name="Thanos" \
  --access-key=thanosaccesskey \
  --secret-key=thanossecretkey

aws s3 mb s3://thanos-metrics \
  --endpoint-url http://rook-ceph-rgw-thanos-store.rook-ceph:80
```

## Configure Thanos Object Store Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objstore-config
  namespace: monitoring
stringData:
  objstore.yml: |
    type: S3
    config:
      bucket: thanos-metrics
      endpoint: rook-ceph-rgw-thanos-store.rook-ceph:80
      region: us-east-1
      access_key: thanosaccesskey
      secret_key: thanossecretkey
      insecure: true
      http_config:
        idle_conn_timeout: 90s
```

## Deploy Thanos Sidecar with Prometheus

Add the Thanos sidecar to the Prometheus pod:

```yaml
containers:
  - name: thanos-sidecar
    image: quay.io/thanos/thanos:v0.36.0
    args:
      - sidecar
      - --tsdb.path=/prometheus
      - --prometheus.url=http://localhost:9090
      - --objstore.config-file=/etc/thanos/objstore.yml
    volumeMounts:
      - name: thanos-config
        mountPath: /etc/thanos
      - name: prometheus-data
        mountPath: /prometheus
volumes:
  - name: thanos-config
    secret:
      secretName: thanos-objstore-config
```

## Deploy Thanos Store Gateway

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install thanos bitnami/thanos \
  --namespace monitoring \
  --set existingObjstoreSecret=thanos-objstore-config \
  --set storegateway.enabled=true \
  --set compactor.enabled=true \
  --set querier.enabled=true
```

## Verify Upload

Check that blocks are being uploaded to Ceph:

```bash
aws s3 ls s3://thanos-metrics/ \
  --endpoint-url http://rook-ceph-rgw-thanos-store.rook-ceph:80 \
  --recursive | head -20
```

## Summary

Ceph RGW serves as an ideal on-premises object store for Thanos, enabling long-term Prometheus metrics retention without cloud storage costs. By using erasure coding on the data pool, you get both durability and storage efficiency for large volumes of historical metrics.
