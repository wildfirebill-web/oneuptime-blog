# How to Use Multus Auto-Discovery with Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Multus, Auto-Discovery, Networking

Description: Learn how to enable and use Multus auto-discovery in Rook-Ceph to automatically detect correct network interfaces for Ceph traffic without manual NAD configuration.

---

Multus auto-discovery in Rook-Ceph allows the operator to automatically identify the appropriate network interfaces on each node, rather than requiring you to manually specify NetworkAttachmentDefinitions. This simplifies configuration for environments where network interfaces have consistent naming across nodes.

## How Multus Auto-Discovery Works

When auto-discovery is enabled, Rook examines the network interfaces on the nodes and attempts to match them against a pattern or criteria you specify. Instead of pinning to a specific NAD by name, Rook uses node network information to create or select appropriate attachments dynamically.

This is useful when:
- Network interface names vary slightly across nodes
- You want Rook to adapt to the available interfaces automatically
- The storage network exists but NADs have not been pre-created

## Enabling Auto-Discovery Mode

Configure the CephCluster to use Multus with auto-discovery:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  network:
    provider: multus
    selectors:
      public: rook-ceph/rook-public-network
      cluster: rook-ceph/rook-cluster-network
    addressRanges:
      public:
      - 192.168.100.0/24
      cluster:
      - 192.168.200.0/24
```

The `addressRanges` field tells Rook which IP ranges correspond to the public and cluster networks. Rook will then select the interface on each node that has an IP in the specified range.

## Using addressRanges for Interface Discovery

The primary auto-discovery mechanism in Rook is `addressRanges`. This is more reliable than name-based NAD selection because it works with any interface naming convention:

```yaml
spec:
  network:
    provider: multus
    addressRanges:
      public:
      - 10.10.0.0/16
      cluster:
      - 10.20.0.0/16
```

Rook checks each node's network interfaces and selects the one with an IP matching the specified range. This works automatically as long as the storage network IPs are pre-configured on the host interfaces.

## Verifying Auto-Discovery Results

After applying the configuration, verify which interfaces Rook has selected:

```bash
# Check operator logs for interface discovery
kubectl -n rook-ceph logs -l app=rook-ceph-operator | grep -i "network\|interface\|discover"
```

```text
discovered public network interface: net1 (192.168.100.15/24)
discovered cluster network interface: net2 (192.168.200.20/24)
```

Check that OSD pods have the correct network annotations:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-osd \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}{"\n"}{end}'
```

## Auto-Discovery with Node Annotations

For more explicit control, annotate nodes to specify which interface to use for Ceph:

```bash
kubectl annotate node node-1 \
  rook.io/public-network-interface=eth1

kubectl annotate node node-1 \
  rook.io/cluster-network-interface=eth2
```

Rook reads these annotations when creating Ceph pods on those nodes, overriding the automatic selection.

## Handling Heterogeneous Nodes

In clusters where different nodes have different network interface names (e.g., `eth1` on some nodes and `ens4` on others), addressRanges-based discovery is the most robust approach:

```yaml
spec:
  network:
    provider: multus
    addressRanges:
      public:
      - 192.168.100.0/24
      - 192.168.101.0/24  # Additional subnet for some nodes
      cluster:
      - 192.168.200.0/24
```

Rook will match any interface with an IP in any of the listed ranges.

## Checking Discovery Status on Nodes

Verify that nodes have the expected interfaces with IPs in the specified ranges:

```bash
for node in node-1 node-2 node-3; do
  echo "=== $node ==="
  ssh $node "ip addr | grep -E '192\.168\.(100|200)'"
done
```

```text
=== node-1 ===
    inet 192.168.100.10/24 brd 192.168.100.255 scope global eth1
    inet 192.168.200.10/24 brd 192.168.200.255 scope global eth2
=== node-2 ===
    inet 192.168.100.11/24 brd 192.168.100.255 scope global eth1
    inet 192.168.200.11/24 brd 192.168.200.255 scope global eth2
```

## Validating Network Connectivity

After auto-discovery configures the network, validate that Ceph daemons can communicate over the discovered interfaces:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

Verify monitor addresses reflect the storage network IPs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon dump | grep "mon\."
```

If monitors show Kubernetes pod network IPs instead of storage network IPs, auto-discovery did not select the correct interfaces. Review the addressRanges configuration and verify the node interfaces have the expected IPs.

## Running the Validation Tool

Rook includes a Multus validation tool that can confirm auto-discovery is working:

```bash
kubectl -n rook-ceph create -f \
  https://raw.githubusercontent.com/rook/rook/main/deploy/examples/multus-validation.yaml
```

Review the validation job output:

```bash
kubectl -n rook-ceph logs job/rook-ceph-multus-validation
```

## Summary

Multus auto-discovery in Rook-Ceph uses `addressRanges` to automatically select the correct network interface on each node based on IP address matching, rather than requiring manual NAD name configuration. Configure the public and cluster network CIDR ranges in the CephCluster spec, and Rook will identify matching interfaces on each node. Use node annotations for explicit per-node overrides when automatic selection does not produce the correct results. Verify auto-discovery by checking operator logs and confirming Ceph monitor addresses use the storage network IPs.
