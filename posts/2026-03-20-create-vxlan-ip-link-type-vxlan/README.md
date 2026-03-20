# How to Create a VXLAN Interface with ip link add type vxlan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, ip command, iproute2, VXLAN, Overlay Network, Networking

Description: Create a VXLAN tunnel interface using ip link add type vxlan to build overlay networks with configurable VNI, local and remote endpoints, and UDP port settings.

## Introduction

`ip link add type vxlan` creates a VXLAN (Virtual Extensible LAN) tunnel endpoint. Each VXLAN interface is a VTEP that encapsulates Ethernet frames in UDP/IP packets. Parameters include VNI (VXLAN Network Identifier), destination port, and local/remote VTEP addresses.

## Create a Basic VXLAN Interface

```bash
# Create a point-to-point VXLAN
ip link add vxlan0 type vxlan \
    id 100 \
    dstport 4789 \
    remote 10.0.0.2 \
    local 10.0.0.1 \
    dev eth0

# Assign an overlay IP
ip addr add 192.168.100.1/24 dev vxlan0

# Bring it up
ip link set vxlan0 up
```

## VXLAN Parameters Explained

```bash
ip link add vxlan0 type vxlan \
    id 100 \         # VNI: 1-16777215
    dstport 4789 \   # UDP destination port (4789 = IANA standard)
    remote 10.0.0.2 \ # Remote VTEP IP (unicast)
    local 10.0.0.1 \  # Local VTEP IP
    dev eth0 \        # Underlay interface
    ttl 64            # Outer IP TTL
```

## VXLAN with Multicast Group

```bash
# Multicast-based VTEP discovery
ip link add vxlan0 type vxlan \
    id 100 \
    dstport 4789 \
    group 239.1.1.1 \
    dev eth0

ip addr add 192.168.100.1/24 dev vxlan0
ip link set vxlan0 up
```

## Verify VXLAN Interface

```bash
# Show detailed VXLAN parameters
ip -d link show vxlan0

# Sample output:
# vxlan id 100 remote 10.0.0.2 local 10.0.0.1 dev eth0 srcport 0 0 dstport 4789 ageing 300

# Show overlay IP
ip addr show vxlan0
```

## Add Static FDB Entries (Head-End Replication)

```bash
# For multicast-less environments, add remote VTEPs manually
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2 permanent
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.3 permanent
```

## Allow VXLAN UDP Traffic

```bash
# Open VXLAN UDP port in firewall
iptables -A INPUT -p udp --dport 4789 -j ACCEPT
iptables -A OUTPUT -p udp --sport 4789 -j ACCEPT
```

## Set Correct MTU

```bash
# VXLAN overhead is ~50 bytes; reduce MTU from underlay's 1500
ip link set vxlan0 mtu 1450
```

## Delete the VXLAN Interface

```bash
ip link set vxlan0 down
ip link delete vxlan0
```

## Remote Host Configuration

```bash
# On the remote host (10.0.0.2)
ip link add vxlan0 type vxlan \
    id 100 dstport 4789 \
    remote 10.0.0.1 local 10.0.0.2 dev eth0
ip addr add 192.168.100.2/24 dev vxlan0
ip link set vxlan0 up
```

## Conclusion

`ip link add type vxlan` creates a VTEP with the specified VNI and endpoint configuration. Add an overlay IP with `ip addr add` and bring up with `ip link set up`. For multi-host setups, use multicast groups or head-end replication via FDB entries. Set MTU to 1450 to avoid fragmentation.
