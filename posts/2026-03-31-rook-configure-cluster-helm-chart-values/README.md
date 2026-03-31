# How to Configure the Rook-Ceph Cluster Helm Chart Values

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Helm, Cluster, Configuration

Description: Learn how to customize the Rook-Ceph cluster Helm chart values to define storage devices, network settings, and Ceph daemon placement.

---

## Overview

The `rook-ceph-cluster` Helm chart deploys a complete Ceph cluster including the CephCluster CR, storage classes, and associated resources. Its `values.yaml` controls every aspect of the Ceph cluster topology.

## Fetching Default Values

```bash
helm show values rook-release/rook-ceph-cluster > rook-cluster-defaults.yaml
```

## Core Cluster Configuration

The most important section is `cephClusterSpec`, which maps directly to the `CephCluster` CR spec:

```yaml
# rook-cluster-values.yaml

cephClusterSpec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.0
    allowUnsupported: false

  dataDirHostPath: /var/lib/rook

  mon:
    count: 3
    allowMultiplePerNode: false

  mgr:
    count: 2
    modules:
      - name: pg_autoscaler
        enabled: true

  dashboard:
    enabled: true
    ssl: true

  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sd[b-z]"
    config:
      osdsPerDevice: "1"
```

## Defining Storage Classes

The chart creates `StorageClass` resources from a list in values:

```yaml
cephBlockPools:
  - name: ceph-blockpool
    spec:
      failureDomain: host
      replicated:
        size: 3
    storageClass:
      enabled: true
      name: ceph-block
      isDefault: true
      reclaimPolicy: Delete
      allowVolumeExpansion: true

cephFileSystems:
  - name: ceph-filesystem
    spec:
      metadataPool:
        replicated:
          size: 3
      dataPools:
        - failureDomain: host
          replicated:
            size: 3
          name: data0
      metadataServer:
        activeCount: 1
        activeStandby: true
    storageClass:
      enabled: true
      name: ceph-filesystem
      pool: data0
      reclaimPolicy: Delete
```

## Enabling Object Storage

```yaml
cephObjectStores:
  - name: ceph-objectstore
    spec:
      metadataPool:
        failureDomain: host
        replicated:
          size: 3
      dataPool:
        failureDomain: host
        erasureCoded:
          dataChunks: 2
          codingChunks: 1
      gateway:
        instances: 1
        port: 80
    storageClass:
      enabled: true
      name: ceph-bucket
      reclaimPolicy: Delete
```

## Installing the Cluster

```bash
helm install rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace rook-ceph \
  -f rook-cluster-values.yaml
```

## Verifying Cluster Status

After deployment, confirm the cluster reaches a healthy state:

```bash
kubectl get cephcluster -n rook-ceph
kubectl -n rook-ceph get pods -l app=rook-ceph-osd
```

## Summary

The `rook-ceph-cluster` Helm chart condenses cluster setup into a single values file covering Ceph version, monitor count, storage device selection, and storage class creation. Keeping this file in source control makes cluster configuration auditable and repeatable across environments.
