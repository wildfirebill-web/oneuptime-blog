# How to Use Ceph RGW for Container Image Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Container Registry, Object Storage, Kubernetes

Description: Learn how to configure Ceph RGW as an S3-compatible backend for container image storage using Harbor or a self-hosted registry.

---

## Introduction

Container images can grow large quickly. Storing them in a dedicated object storage backend like Ceph RGW keeps your registry scalable, cost-effective, and independent of cloud vendor lock-in. This guide walks through connecting a container registry to Ceph RGW.

## Prerequisites

- A running Rook-Ceph cluster with RGW enabled
- `kubectl` and `radosgw-admin` access
- Harbor or Docker Registry v2 deployed in Kubernetes

## Setting Up the RGW Object Store

First, create a CephObjectStore and expose it via a service:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: registry-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  gateway:
    port: 80
    instances: 2
```

Apply and wait for the store to become ready:

```bash
kubectl apply -f objectstore.yaml
kubectl -n rook-ceph get cephobjectstore registry-store
```

## Creating a Dedicated User for the Registry

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
    --uid=registry-user \
    --display-name="Container Registry User" \
    --access-key=REGACCESSKEY \
    --secret-key=REGSECRETKEY
```

Create an S3 bucket for images:

```bash
aws s3api create-bucket \
  --bucket container-images \
  --endpoint-url http://rook-ceph-rgw-registry-store.rook-ceph:80
```

## Configuring Harbor to Use Ceph RGW

In Harbor's `values.yaml` when deploying via Helm, set the storage backend:

```yaml
persistence:
  imageChartStorage:
    type: s3
    s3:
      region: us-east-1
      bucket: container-images
      accesskey: REGACCESSKEY
      secretkey: REGSECRETKEY
      regionendpoint: http://rook-ceph-rgw-registry-store.rook-ceph:80
      encrypt: false
      secure: false
```

Apply the Helm chart:

```bash
helm upgrade --install harbor harbor/harbor \
  -f values.yaml \
  --namespace harbor
```

## Configuring Docker Registry v2

For a plain Docker Registry, create a config file:

```yaml
version: 0.1
storage:
  s3:
    accesskey: REGACCESSKEY
    secretkey: REGSECRETKEY
    region: us-east-1
    regionendpoint: http://rook-ceph-rgw-registry-store.rook-ceph:80
    bucket: container-images
    encrypt: false
    secure: false
    v4auth: true
http:
  addr: :5000
```

Mount this as a ConfigMap and deploy the registry pod referencing it.

## Testing the Setup

Push a test image:

```bash
docker tag nginx:latest my-registry.example.com/nginx:latest
docker push my-registry.example.com/nginx:latest
```

Verify the image layers are stored in Ceph:

```bash
aws s3 ls s3://container-images/ \
  --endpoint-url http://rook-ceph-rgw-registry-store.rook-ceph:80 \
  --recursive | head -20
```

## Summary

Ceph RGW provides an S3-compatible backend for container image registries, eliminating dependency on cloud object storage. By creating a dedicated RGW user and bucket and pointing Harbor or Docker Registry at the RGW endpoint, you can host container images entirely on-premises with Rook-managed Ceph storage.
