# How to Configure Read Affinity with CRUSH Location Labels in Rook CSI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Csi, Read Affinity, Crush, Performance, Kubernetes

Description: Learn how to configure read affinity using CRUSH location labels in Rook CSI to direct read I/O to the nearest OSD replica for improved latency.

---

## Overview

Read affinity allows the Rook CSI driver to prefer reading from OSD replicas that are topologically close to the requesting pod. By adding CRUSH location labels to Kubernetes nodes and configuring the CSI driver accordingly, you can reduce cross-rack or cross-AZ read latency for RBD volumes.

## How Read Affinity Works

When a pod requests a read from an RBD volume:
1. Without read affinity - the CRUSH algorithm selects any replica
2. With read affinity - the CSI driver prefers replicas on OSDs located in the same CRUSH bucket (e.g., same rack or AZ) as the requesting node

## Step 1 - Label Kubernetes Nodes with CRUSH Location

Add CRUSH location labels to nodes that indicate their physical placement:

```bash
# Label nodes in rack1
kubectl label node node1 topology.rook.io/crush-root=default
kubectl label node node1 topology.rook.io/crush-datacenter=dc1
kubectl label node node1 topology.rook.io/crush-rack=rack1

kubectl label node node2 topology.rook.io/crush-root=default
kubectl label node node2 topology.rook.io/crush-datacenter=dc1
kubectl label node node2 topology.rook.io/crush-rack=rack1

# Label nodes in rack2
kubectl label node node3 topology.rook.io/crush-root=default
kubectl label node node3 topology.rook.io/crush-datacenter=dc1
kubectl label node node3 topology.rook.io/crush-rack=rack2
```

## Step 2 - Configure CRUSH Location on OSDs

Update the CephCluster to add CRUSH location labels to OSDs:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18
  storage:
    useAllNodes: false
    nodes:
      - name: node1
        config:
          crushLocation: "root=default datacenter=dc1 rack=rack1"
      - name: node2
        config:
          crushLocation: "root=default datacenter=dc1 rack=rack1"
      - name: node3
        config:
          crushLocation: "root=default datacenter=dc1 rack=rack2"
```

## Step 3 - Enable Read Affinity in the CSI ConfigMap

Configure the CSI config to enable read affinity:

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
          "10.0.0.1:6789",
          "10.0.0.2:6789",
          "10.0.0.3:6789"
        ],
        "readAffinity": {
          "enabled": true,
          "crushLocationLabels": [
            "topology.rook.io/crush-rack"
          ]
        }
      }
    ]
```

Apply the config:

```bash
kubectl apply -f csi-config.yaml
```

## Step 4 - Restart CSI Node Plugins

For the changes to take effect, restart the CSI node DaemonSet:

```bash
kubectl -n rook-ceph rollout restart daemonset rook-ceph-csi-rbdplugin
```

Verify the DaemonSet is updated:

```bash
kubectl -n rook-ceph rollout status daemonset rook-ceph-csi-rbdplugin
```

## Step 5 - Verify Read Affinity is Working

Check that the CSI node plugin picked up the CRUSH location from node labels:

```bash
kubectl -n rook-ceph logs daemonset/rook-ceph-csi-rbdplugin -c csi-rbdplugin | grep -i "crush"
```

Test I/O and check which OSD is serving reads:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd perf
```

## Using Multiple CRUSH Location Labels

For finer-grained affinity, specify multiple labels in priority order:

```yaml
"readAffinity": {
  "enabled": true,
  "crushLocationLabels": [
    "topology.rook.io/crush-host",
    "topology.rook.io/crush-rack",
    "topology.rook.io/crush-datacenter"
  ]
}
```

The CSI driver tries to match the most specific location first, falling back to less specific matches.

## Summary

Configuring read affinity with CRUSH location labels in Rook CSI involves labeling Kubernetes nodes with their physical topology, configuring OSD CRUSH locations to match, and enabling read affinity in the CSI config with the relevant label keys. Once configured, the RBD CSI node plugin directs read I/O to the nearest OSD replica, reducing latency for topology-aware clusters spanning multiple racks or availability zones.
