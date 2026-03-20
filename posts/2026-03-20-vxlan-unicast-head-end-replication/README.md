# How to Configure VXLAN with Unicast Head-End Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VXLAN, Unicast, Head-End Replication, Overlay Network, Networking, SDN

Description: Configure VXLAN unicast head-end replication to flood BUM traffic without multicast by manually adding remote VTEP addresses to the VXLAN forwarding database.

## Introduction

Unicast head-end replication (HER) is an alternative to multicast VXLAN that works in environments without multicast routing. Instead of flooding to a multicast group, the originating VTEP sends individual copies of BUM frames to each known remote VTEP. Remote VTEP IPs are statically added to the VXLAN FDB.

## Prerequisites

- VXLAN interfaces created on each host
- Knowledge of all remote VTEP underlay IPs
- Root access on all hosts

## Create VXLAN Without Multicast

```bash
# Create VXLAN without a multicast group (unicast mode)
# No 'group' or 'remote' parameter — start with empty FDB
ip link add vxlan0 type vxlan \
    id 100 \
    dstport 4789 \
    local 10.0.0.1 \
    dev eth0

ip addr add 10.100.0.1/24 dev vxlan0
ip link set vxlan0 up
```

## Add Remote VTEP Entries to FDB

The zero MAC (`00:00:00:00:00:00`) entry with a destination IP acts as the flood entry — unknown unicast, broadcast, and multicast frames go to that VTEP:

```bash
# Add each remote VTEP as a flood entry
# These are permanent entries for BUM replication
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2 permanent
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.3 permanent

# Note: use 'append' not 'add' for multiple entries with same zero-MAC
```

## Full Three-Host Setup

### Host 1 (10.0.0.1, overlay: 10.100.0.1)

```bash
ip link add vxlan0 type vxlan id 100 dstport 4789 local 10.0.0.1 dev eth0
ip addr add 10.100.0.1/24 dev vxlan0
ip link set vxlan0 up
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2 permanent
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.3 permanent
```

### Host 2 (10.0.0.2, overlay: 10.100.0.2)

```bash
ip link add vxlan0 type vxlan id 100 dstport 4789 local 10.0.0.2 dev eth0
ip addr add 10.100.0.2/24 dev vxlan0
ip link set vxlan0 up
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.1 permanent
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.3 permanent
```

### Host 3 (10.0.0.3, overlay: 10.100.0.3)

```bash
ip link add vxlan0 type vxlan id 100 dstport 4789 local 10.0.0.3 dev eth0
ip addr add 10.100.0.3/24 dev vxlan0
ip link set vxlan0 up
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.1 permanent
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2 permanent
```

## Verify FDB Configuration

```bash
# Show VXLAN FDB entries
bridge fdb show dev vxlan0

# Sample output:
# 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2 self permanent
# 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.3 self permanent
# aa:bb:cc:dd:ee:11 dev vxlan0 dst 10.0.0.2 self   <- dynamically learned
```

## Test Connectivity

```bash
# Test VXLAN overlay connectivity
ping -c 3 10.100.0.2
ping -c 3 10.100.0.3
```

## Multicast vs Head-End Replication

| Aspect | Multicast | Head-End Replication |
|---|---|---|
| Requirement | Multicast routing | No special underlay needed |
| Scaling | Better (one flood copy) | Worse (N-1 copies per BUM) |
| Configuration | Simple (one group) | Manual VTEP entries |
| Use case | On-premises | Cloud/overlays |

## Conclusion

VXLAN unicast head-end replication works in any environment without requiring multicast routing support. Manually add each remote VTEP as a zero-MAC flood entry using `bridge fdb append`. As the network grows, add new VTEP entries on all existing hosts. For dynamic VTEP discovery, use a control plane like BGP EVPN or a network overlay controller.
