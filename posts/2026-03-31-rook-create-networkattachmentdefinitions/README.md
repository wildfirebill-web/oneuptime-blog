# How to Create NetworkAttachmentDefinitions for Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Multus, NetworkAttachmentDefinition, Networking

Description: Learn how to create and configure NetworkAttachmentDefinitions for Rook-Ceph to separate storage traffic onto dedicated network interfaces using Multus.

---

NetworkAttachmentDefinitions (NADs) are Kubernetes custom resources provided by the Multus CNI plugin. They define additional network interfaces that pods can attach to. For Rook-Ceph, NADs are used to direct Ceph storage traffic (OSD replication, client I/O) onto a dedicated storage network, separate from the Kubernetes pod network.

## Prerequisites

Multus CNI must be installed and configured as a meta-CNI plugin in your cluster. Verify Multus is running:

```bash
kubectl -n kube-system get pods | grep multus
```

```text
kube-multus-ds-abc12   1/1     Running   0          5d
```

Also verify the NetworkAttachmentDefinition CRD is present:

```bash
kubectl get crd | grep network-attachment-definitions
```

## Understanding NAD Types for Rook

Rook supports two types of NADs for different Ceph networks:

- **Public network NAD**: Used for client-to-Ceph communication (reads, writes, monitor connections)
- **Cluster network NAD**: Used for OSD-to-OSD replication traffic

Separating these onto dedicated interfaces provides:
- Higher throughput for storage operations
- Isolation of storage traffic from application traffic
- Better security posture

## Creating a Macvlan NAD

The most common NAD type for Rook is Macvlan, which creates virtual network interfaces on a physical NIC:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rook-public-network
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24",
        "range_start": "192.168.100.10",
        "range_end": "192.168.100.200"
      }
    }
```

Apply the NAD:

```bash
kubectl apply -f rook-public-network-nad.yaml
```

Create a second NAD for the cluster network:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rook-cluster-network
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth2",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.200.0/24",
        "range_start": "192.168.200.10",
        "range_end": "192.168.200.200"
      }
    }
```

```bash
kubectl apply -f rook-cluster-network-nad.yaml
```

## Creating an IPVLAN NAD

For environments where Macvlan cannot be used (some cloud providers or where traffic must share the same MAC address), IPVLAN is an alternative:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rook-public-network
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "ipvlan",
      "master": "eth1",
      "mode": "l2",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24"
      }
    }
```

## Creating a Host-Device NAD

For direct access to a specific physical NIC without virtualization:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rook-cluster-network
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "host-device",
      "device": "eth2",
      "ipam": {
        "type": "static",
        "addresses": [
          {
            "address": "192.168.200.10/24"
          }
        ]
      }
    }
```

## Verifying NAD Creation

After creating NADs, verify they are properly registered:

```bash
kubectl -n rook-ceph get network-attachment-definitions
```

```text
NAME                   AGE
rook-public-network    2m
rook-cluster-network   2m
```

Inspect a NAD to verify configuration:

```bash
kubectl -n rook-ceph get network-attachment-definition rook-public-network -o yaml
```

## Referencing NADs in Rook CephCluster

Once NADs are created, reference them in the CephCluster CR:

```yaml
spec:
  network:
    provider: multus
    selectors:
      public: rook-ceph/rook-public-network
      cluster: rook-ceph/rook-cluster-network
```

The format is `<namespace>/<nad-name>`. Rook will annotate Ceph pods with the Multus annotation to attach the specified network interfaces.

## Checking Pod Network Annotations

After deploying with Multus, verify that Ceph pods have the correct network annotations:

```bash
kubectl -n rook-ceph get pod rook-ceph-osd-0-xxx \
  -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/networks}'
```

```text
rook-ceph/rook-public-network, rook-ceph/rook-cluster-network
```

## Summary

NetworkAttachmentDefinitions for Rook-Ceph are created in the `rook-ceph` namespace using the NAD CRD provided by Multus. Create separate NADs for the public network (client traffic) and cluster network (OSD replication) using Macvlan, IPVLAN, or host-device type depending on your environment. Reference the NADs in the CephCluster CR under `spec.network.selectors`. Verify pod annotations after deployment to confirm Multus is correctly attaching the secondary network interfaces.
