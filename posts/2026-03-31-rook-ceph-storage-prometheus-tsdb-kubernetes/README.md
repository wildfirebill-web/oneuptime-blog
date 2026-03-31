# How to Set Up Ceph Storage for Prometheus TSDB on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Prometheus, Kubernetes, Storage, TSDB, Monitoring

Description: Learn how to back Prometheus time-series data with Ceph RBD block storage on Kubernetes using Rook for durable, scalable TSDB persistence.

---

## Overview

Prometheus stores time-series metrics in its built-in TSDB (Time Series Database), a write-heavy workload that demands reliable block storage. By using Rook-Ceph RBD volumes, you give Prometheus durable storage that survives pod evictions and node failures.

## Create the CephBlockPool and StorageClass

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: prometheus-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-prometheus
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: prometheus-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
allowVolumeExpansion: true
```

## Deploy Prometheus with Ceph-Backed Storage

Using the kube-prometheus-stack Helm chart, override the storage class:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=ceph-rbd-prometheus \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

Alternatively, configure via values file:

```yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: ceph-rbd-prometheus
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 50Gi
```

## Verify TSDB Persistence

Check the PVC is bound:

```bash
kubectl -n monitoring get pvc
# prometheus-kube-prometheus-prometheus-db-... Bound  50Gi  RWO  ceph-rbd-prometheus
```

Verify TSDB head stats via the Prometheus API:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-prometheus 9090
curl http://localhost:9090/api/v1/status/tsdb | jq '.data.headStats'
```

## Performance Considerations

Enable write-ahead log (WAL) compression to reduce I/O:

```yaml
prometheus:
  prometheusSpec:
    walCompression: true
```

Use `rbd_cache_writethrough_until_flush = true` on the Ceph side to prevent stale reads after crashes.

## Monitoring Ceph Pool Health

```bash
ceph osd pool stats prometheus-pool
ceph df detail | grep prometheus
```

## Summary

Pairing Prometheus TSDB with Ceph RBD on Kubernetes ensures metrics survive pod restarts and Kubernetes upgrades. The dedicated BlockPool isolates Prometheus I/O from other workloads, and volume expansion lets you grow storage without downtime as your metrics volume grows.
