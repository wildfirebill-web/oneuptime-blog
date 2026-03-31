# How to Optimize Ceph Network Costs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Network, Cost Optimization, Bandwidth, Infrastructure

Description: Reduce Ceph network infrastructure costs by separating public and cluster networks, right-sizing bandwidth, enabling network compression, and optimizing traffic patterns.

---

## Network as a Cost Driver

Ceph generates significant network traffic for replication, recovery, and client I/O. Network infrastructure - switches, NICs, cabling, and bandwidth - can represent 20-30% of total cluster hardware cost if not planned carefully.

## Separate Public and Cluster Networks

Separating networks lets you use cheaper hardware for cluster (replication) traffic and dedicate faster links to client-facing I/O:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  network:
    provider: host
    selectors:
      public: "enp1s0"    # Client-facing, 10 GbE
      cluster: "enp2s0"   # Replication, 10 GbE (can be separate VLAN)
```

## Right-Size Network Bandwidth

Avoid over-buying NICs by understanding actual traffic patterns:

```bash
# Monitor current network utilization per OSD
ceph daemon osd.0 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
sent = d['osd']['osd_network_sent_bytes']
recv = d['osd']['osd_network_received_bytes']
print(f'Sent: {sent/(1024**2):.1f} MB, Recv: {recv/(1024**2):.1f} MB')
"

# If peak utilization is < 40% of link capacity, you may not need 25 GbE
# 10 GbE can handle ~1.2 GB/s sustained, sufficient for most HDDs
```

## Enable Messenger v2 with Compression

```bash
# Enable network-level compression for cluster traffic
ceph config set global ms_compress_on_wire true

# This reduces replication bandwidth by 20-40% for compressible data
# Check messenger stats
ceph daemon osd.0 perf dump | grep ms_
```

## Limit Recovery Bandwidth During Business Hours

```bash
# Throttle recovery traffic to avoid saturating client-facing links
ceph config set osd osd_recovery_max_active 2
ceph config set osd osd_max_backfills 1

# Set recovery bandwidth limit (bytes per second)
ceph tell osd.* injectargs '--osd-recovery-sleep-hdd 0.1'

# Restore full speed during maintenance windows
ceph config set osd osd_recovery_max_active 5
```

## Use Jumbo Frames for Cluster Network

Jumbo frames reduce CPU overhead and improve throughput efficiency:

```bash
# Enable 9000 MTU on cluster network interfaces
# /etc/network/interfaces or via NetworkManager:
ip link set enp2s0 mtu 9000

# Verify end-to-end MTU works
ping -M do -s 8972 <osd-node-ip>

# Configure Ceph to use jumbo frames
ceph config set global ms_tcp_rcvbuf 4194304
```

## Optimize for Local-Rack Traffic

Configure CRUSH to prefer same-rack placement to minimize top-of-rack switch hops:

```bash
# Create rack-aware CRUSH rule
ceph osd crush rule create-replicated rack-rule default rack host

# This reduces inter-rack traffic (which often crosses more expensive switches)
ceph osd pool set mypool crush_rule rack-rule
```

## Calculate Network Hardware Savings

```bash
# Scenario: 20-node cluster
# All 25 GbE: 20 nodes x $300/NIC = $6,000 in NICs
#             1x 25G switch: $15,000
# Total: $21,000

# 10 GbE alternative (if utilization < 40%):
# 20 nodes x $80/NIC = $1,600 in NICs
# 1x 10G switch: $3,000
# Total: $4,600
# Savings: $16,400
```

## Summary

Ceph network costs can be significantly reduced by right-sizing NIC and switch hardware based on actual traffic patterns, enabling messenger compression to cut bandwidth consumption, and separating public/cluster networks to allow independent scaling. For most HDD-based clusters, 10 GbE per node is sufficient and saves thousands in infrastructure costs.
