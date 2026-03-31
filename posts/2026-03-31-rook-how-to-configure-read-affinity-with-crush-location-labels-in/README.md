# How to Configure Read Affinity with CRUSH Location Labels in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Read Affinity, Crush, Kubernetes, Performance

Description: Configure Rook CSI read affinity using Kubernetes node CRUSH location labels to route RBD reads to the nearest OSD for improved latency and throughput.

---

## Overview

By default, Ceph RBD reads are served by the primary OSD regardless of which Kubernetes node the pod is on. Read affinity allows the CSI driver to route read requests to OSDs located on or near the same node as the pod, reducing network hops and improving read latency. This is especially beneficial in stretched or multi-zone clusters.

## How Read Affinity Works

The Rook CSI driver uses CRUSH location labels on Kubernetes nodes to select a replica that is topologically close to the client. If a replica exists on the local node or in the same rack/zone, reads are served from there instead of going to the primary OSD on a potentially distant node.

## Step 1 - Label Kubernetes Nodes with CRUSH Locations

Label each node with its CRUSH topology:

```bash
# Single-zone cluster - label by host
kubectl label node worker-01 topology.rook.io/crush-location='{"host":"worker-01"}'
kubectl label node worker-02 topology.rook.io/crush-location='{"host":"worker-02"}'
kubectl label node worker-03 topology.rook.io/crush-location='{"host":"worker-03"}'
```

For a multi-zone cluster:

```bash
# Zone-aware labels
kubectl label node worker-01 topology.rook.io/crush-location='{"host":"worker-01","zone":"zone-a"}'
kubectl label node worker-02 topology.rook.io/crush-location='{"host":"worker-02","zone":"zone-a"}'
kubectl label node worker-03 topology.rook.io/crush-location='{"host":"worker-03","zone":"zone-b"}'
```

## Step 2 - Enable Read Affinity in Rook CSI Config

Update the Rook CSI configuration ConfigMap to enable read affinity:

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
        "clusterID": "rook-ceph",
        "monitors": [
          "192.168.1.10:6789",
          "192.168.1.11:6789",
          "192.168.1.12:6789"
        ],
        "readAffinity": {
          "enabled": true,
          "crushLocationLabels": [
            "topology.rook.io/crush-location"
          ]
        }
      }
    ]
```

Apply the ConfigMap:

```bash
kubectl apply -f csi-config.yaml
```

## Step 3 - Restart CSI Nodeplugin Pods

After updating the config, restart the CSI node plugin DaemonSet:

```bash
kubectl -n rook-ceph rollout restart daemonset/csi-rbdplugin
```

## Step 4 - Verify Read Affinity Configuration

Check the node plugin logs to verify read affinity is active:

```bash
kubectl -n rook-ceph logs daemonset/csi-rbdplugin -c csi-rbdplugin | grep -i "read affinity"
```

Expected log line:

```text
read affinity enabled for cluster rook-ceph with CRUSH location labels [topology.rook.io/crush-location]
```

## Step 5 - Verify OSD CRUSH Location Matches

Ensure OSDs have CRUSH locations that match your node labels:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd tree
```

The CRUSH tree should reflect the same topology you labeled on the nodes:

```text
ID  CLASS  WEIGHT   TYPE NAME          STATUS  REWEIGHT  PRI-AFF
-1         0.17999  root default
-3         0.05999      zone zone-a
-5         0.05999          host worker-01
 0    hdd  0.01999              osd.0      up   1.00000   1.00000
```

## Performance Considerations

Read affinity works best when:

- Your cluster has replicated pools (not erasure coded)
- The replication factor matches or exceeds the number of zones/hosts
- Network latency between zones/racks is significant

Monitor read latency before and after enabling read affinity:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd perf | grep read_latency
```

## Summary

Read affinity in Rook CSI routes RBD reads to topologically close OSDs by matching Kubernetes node CRUSH location labels against the Ceph CRUSH map. Label nodes with `topology.rook.io/crush-location`, enable `readAffinity` in the CSI config, and restart the node plugin. This reduces cross-zone or cross-rack read latency, particularly in multi-zone Kubernetes deployments where pods and their storage replicas may be co-located.
