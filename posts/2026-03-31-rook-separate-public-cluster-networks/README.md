# How to Separate Public and Cluster Networks in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Networking, Architecture, Performance, Operation

Description: Configure separate public and cluster networks in Ceph to isolate client traffic from OSD replication traffic, improving performance, security, and recovery speed.

---

## Why Separate Networks?

A single-network Ceph deployment means client I/O, OSD-to-OSD replication, and recovery traffic all compete on the same interface. When an OSD fails and recovery generates 10 GB/s of inter-OSD traffic, clients on the same network suffer degraded performance.

Two dedicated networks solve this:
- **Public network**: Client-to-MON and client-to-OSD traffic
- **Cluster network**: OSD-to-OSD replication, recovery, heartbeat, scrub traffic

## Network Architecture

Typical network layout for a production Ceph cluster:

```text
Clients/Applications
       |
   10 GbE (public: 10.0.1.0/24)
       |
  Ceph OSD nodes
       |
  25 GbE (cluster: 192.168.10.0/24)
       |
  OSD-to-OSD replication
```

Each OSD node needs two NICs or VLAN-separated interfaces.

## Configuring in Traditional Ceph (ceph.conf)

```ini
[global]
public_network = 10.0.1.0/24
cluster_network = 192.168.10.0/24

[osd]
# Bind OSD to both interfaces
ms_bind_ipv4 = true
```

Apply and verify:

```bash
ceph config set mon public_network 10.0.1.0/24
ceph config set osd cluster_network 192.168.10.0/24

# Verify OSDs are using correct network
ceph osd dump | grep -E "public_addr|cluster_addr"
```

## Configuring in Rook with Host Networking

For Rook using host networking mode:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  network:
    provider: host
    addressRanges:
      public:
      - "10.0.1.0/24"
      cluster:
      - "192.168.10.0/24"
```

## Configuring in Rook with Multus

For production Kubernetes deployments, use Multus for proper network isolation:

```bash
# Install Multus CNI
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml
```

Create NetworkAttachmentDefinition for each Ceph network:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ceph-public
  namespace: rook-ceph
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "10.0.1.0/24"
      }
    }
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ceph-cluster
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
        "range": "192.168.10.0/24"
      }
    }
```

Reference them in the CephCluster:

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
      public: rook-ceph/ceph-public
      cluster: rook-ceph/ceph-cluster
```

## Verifying Network Separation

Confirm OSDs are bound to correct interfaces:

```bash
# Check OSD public and cluster addresses
ceph osd dump | grep -E "osd\.[0-9]+ " | awk '{print $1, $NF}' | head -10

# Verify cluster network traffic goes to correct interface
tcpdump -i eth1 -n port 6800:7300 | head -20

# Check OSD messenger stats
ceph daemon osd.0 perf dump | grep msgr
```

## Performance Impact

Measure improvement with iperf3 before and after separation:

```bash
# Measure public network capacity
iperf3 -s -B 10.0.1.1  # Server on public network
iperf3 -c 10.0.1.1 -P 4 -t 30  # Client

# Measure cluster network capacity
iperf3 -s -B 192.168.10.1  # Server on cluster network
iperf3 -c 192.168.10.1 -P 4 -t 30  # Client
```

During recovery, check that public network throughput is unaffected:

```bash
# Monitor both interfaces during recovery
sar -n DEV 1 30 | grep -E "eth0|eth1"
```

## Summary

Separating Ceph public and cluster networks prevents OSD replication and recovery traffic from impacting client performance. In Rook Kubernetes environments, Multus CNI provides true network-level isolation by binding different pods to different physical interfaces. This architecture is especially valuable during cluster recovery events, where inter-OSD traffic can otherwise saturate shared network infrastructure and degrade client operations.
