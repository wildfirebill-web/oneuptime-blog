# How to Configure CSI Read Affinity in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CSI, Read Affinity, Kubernetes, Storage, Performance

Description: Configure CSI read affinity in Rook-Ceph so that RBD and CephFS reads prefer OSDs on the same node as the workload, reducing network overhead.

---

## What Is CSI Read Affinity

By default, Ceph serves reads from the primary OSD regardless of where the client pod is running. CSI read affinity is a feature that instructs the Ceph CSI driver to prefer reading from a replica OSD that is local to the Kubernetes node running the workload. This reduces cross-node network traffic for read-heavy workloads without affecting data durability or write behavior.

Read affinity is configured at the CSI driver level through a ConfigMap that the Rook operator manages.

## Prerequisites

- Rook-Ceph v1.10 or later
- Ceph Pacific (v16) or later
- The Ceph RBD or CephFS CSI driver deployed by Rook

## Enabling Read Affinity via the Rook ConfigMap

Rook exposes CSI configuration through the `rook-ceph-csi-config` ConfigMap in the `rook-ceph` namespace. To enable read affinity, edit or patch this ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-ceph-csi-config
  namespace: rook-ceph
data:
  config.json: |
    [
      {
        "clusterID": "<your-cluster-fsid>",
        "monitors": [
          "10.0.0.1:6789",
          "10.0.0.2:6789",
          "10.0.0.3:6789"
        ],
        "readAffinity": {
          "enabled": true,
          "crushLocationLabels": [
            "topology.kubernetes.io/zone",
            "kubernetes.io/hostname"
          ]
        }
      }
    ]
```

Replace `<your-cluster-fsid>` with the output of:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fsid
```

## Understanding crushLocationLabels

The `crushLocationLabels` array maps Kubernetes node labels to CRUSH topology keys. When a CSI node plugin starts on a node, it reads these labels and passes them as CRUSH location hints to the Ceph client. Ceph then prefers replicas in the same CRUSH bucket.

Common label mappings:

| Kubernetes Label | CRUSH Key |
|---|---|
| `topology.kubernetes.io/zone` | zone |
| `topology.kubernetes.io/region` | region |
| `kubernetes.io/hostname` | host |

Ensure your nodes have the correct labels applied:

```bash
kubectl label node worker-01 topology.kubernetes.io/zone=zone-a
kubectl label node worker-02 topology.kubernetes.io/zone=zone-b
```

## Applying the Configuration

After editing the ConfigMap, restart the CSI node plugins to pick up the new configuration:

```bash
kubectl -n rook-ceph rollout restart daemonset/csi-rbdplugin
kubectl -n rook-ceph rollout restart daemonset/csi-cephfsplugin
```

Wait for the rollout to complete:

```bash
kubectl -n rook-ceph rollout status daemonset/csi-rbdplugin
```

## Verifying Read Affinity Is Active

Check the CSI node plugin logs for read affinity messages:

```bash
kubectl -n rook-ceph logs daemonset/csi-rbdplugin -c csi-rbdplugin | grep -i "affinity"
```

You can also run a benchmark pod on a specific node and use `ceph tell` to observe which OSD services the reads, confirming they come from a local replica.

## Limitations and Considerations

- Read affinity only applies when a local replica exists. If the primary is local, reads are already optimal.
- For erasure-coded pools, read affinity applies to the primary shard only; EC reads always involve multiple OSDs.
- Enabling read affinity adds a small latency overhead for OSD selection during the read path. For write-heavy workloads the benefit is negligible.

## Summary

CSI read affinity in Rook-Ceph routes reads to the OSD closest to the consuming pod by mapping Kubernetes node topology labels to Ceph CRUSH locations. Configure it through the `rook-ceph-csi-config` ConfigMap, restart the CSI node daemonsets, and verify that node topology labels are present. This is a low-risk optimization that meaningfully reduces cross-node read traffic in multi-zone or large single-site clusters.
