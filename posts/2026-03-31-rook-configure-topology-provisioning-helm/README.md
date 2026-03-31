# How to Configure Topology Provisioning in Rook Helm Chart

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Helm, Topology, Storage

Description: Configure topology-aware provisioning in the Rook-Ceph Helm chart to ensure volumes are created on nodes matching the workload topology.

---

## Overview

Topology-aware provisioning ensures that PersistentVolumes are created in the same zone or region as the pods that will consume them. This is critical in multi-zone clusters to avoid cross-zone latency and reduce network costs. Rook-Ceph supports topology-aware provisioning through CSI topology features.

## Enabling Topology in the Helm Chart

The topology feature is controlled in the operator chart values:

```yaml
csi:
  topology:
    enabled: true
    domainLabels:
      - topology.kubernetes.io/zone
      - topology.kubernetes.io/region
```

`domainLabels` specifies which node labels define topology boundaries. Kubernetes standard zone and region labels are the most common choices.

## Node Labeling

Ensure nodes carry the topology labels that match your `domainLabels` configuration:

```bash
kubectl label node worker-node-1 topology.kubernetes.io/zone=zone-a
kubectl label node worker-node-2 topology.kubernetes.io/zone=zone-a
kubectl label node worker-node-3 topology.kubernetes.io/zone=zone-b
```

## StorageClass with Topology Binding

Configure the StorageClass to use `WaitForFirstConsumer` binding mode so the provisioner knows which zone the pod needs before creating the volume:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-topology
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.kubernetes.io/zone
        values:
          - zone-a
          - zone-b
```

## Applying the Helm Configuration

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  -f rook-topology-values.yaml
```

Verify topology is reflected in the CSI driver registration:

```bash
kubectl get csinode <node-name> -o yaml | grep topology
```

## Testing Topology Binding

Deploy a pod with a zone-specific node selector and create a PVC using the topology StorageClass. The PVC should remain `Pending` until the pod is scheduled, then bind to a volume on the same zone:

```bash
kubectl describe pvc my-topology-pvc | grep -A5 "Events"
```

## Summary

Topology-aware provisioning in Rook-Ceph requires enabling the `csi.topology` feature in the Helm chart, labeling nodes with zone information, and configuring StorageClasses with `WaitForFirstConsumer` binding mode. This ensures volumes are always co-located with the pods using them, eliminating cross-zone data access penalties.
