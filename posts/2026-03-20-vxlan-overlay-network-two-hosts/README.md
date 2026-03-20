# How to Create a VXLAN Overlay Network Between Two Hosts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VXLAN, Overlay Network, Networking, Layer 2, Containers, SDN

Description: Build a complete VXLAN overlay network between two Linux hosts, connecting them at Layer 2 with a shared virtual LAN segment over any IP underlay.

## Introduction

A VXLAN overlay network creates a virtual LAN that spans two or more physical hosts over any IP network. Hosts on the overlay see each other as if they are on the same local network, regardless of the physical topology. This guide builds a complete two-host overlay.

## Network Plan

```text
Underlay: Host A = 10.0.0.1, Host B = 10.0.0.2
VXLAN VNI: 1000
Overlay: Host A bridge = 10.200.0.1/24, Host B bridge = 10.200.0.2/24
```

## Complete Setup on Host A

```bash
#!/bin/bash
# setup-vxlan-host-a.sh

UNDERLAY_IP=10.0.0.1
REMOTE_IP=10.0.0.2
VNI=1000
OVERLAY_IP=10.200.0.1/24

# Create VXLAN interface

ip link add vxlan${VNI} type vxlan \
    id ${VNI} \
    dstport 4789 \
    local ${UNDERLAY_IP} \
    dev eth0
ip link set vxlan${VNI} up

# Create bridge
ip link add br-overlay type bridge
ip link set br-overlay type bridge stp_state 0
ip link set br-overlay up

# Attach VXLAN to bridge
ip link set vxlan${VNI} master br-overlay

# Assign overlay IP to bridge
ip addr add ${OVERLAY_IP} dev br-overlay

# Add remote VTEP for BUM flooding
bridge fdb append 00:00:00:00:00:00 dev vxlan${VNI} dst ${REMOTE_IP} permanent

echo "Host A VXLAN overlay configured"
ip addr show br-overlay
bridge fdb show dev vxlan${VNI}
```

## Complete Setup on Host B

```bash
#!/bin/bash
# setup-vxlan-host-b.sh

UNDERLAY_IP=10.0.0.2
REMOTE_IP=10.0.0.1
VNI=1000
OVERLAY_IP=10.200.0.2/24

ip link add vxlan${VNI} type vxlan \
    id ${VNI} \
    dstport 4789 \
    local ${UNDERLAY_IP} \
    dev eth0
ip link set vxlan${VNI} up

ip link add br-overlay type bridge
ip link set br-overlay type bridge stp_state 0
ip link set br-overlay up

ip link set vxlan${VNI} master br-overlay
ip addr add ${OVERLAY_IP} dev br-overlay

bridge fdb append 00:00:00:00:00:00 dev vxlan${VNI} dst ${REMOTE_IP} permanent

echo "Host B VXLAN overlay configured"
```

## Allow VXLAN Traffic

```bash
# Allow VXLAN UDP (run on both hosts)
iptables -A INPUT -p udp --dport 4789 -j ACCEPT
```

## Test the Overlay

```bash
# From Host A, ping Host B's overlay IP
ping -c 3 10.200.0.2

# Traceroute - should show direct VXLAN path
traceroute 10.200.0.2

# Check MAC address learning via VXLAN
bridge fdb show dev vxlan1000 | grep -v permanent
```

## Monitor VXLAN Traffic

```bash
# Watch VXLAN encapsulated traffic
tcpdump -i eth0 udp port 4789 -n -v

# View inner frames on the bridge
tcpdump -i br-overlay -n
```

## Conclusion

A complete VXLAN overlay between two hosts requires: a VXLAN interface with matching VNI on both hosts, a bridge with the VXLAN attached, an overlay IP on the bridge, and FDB flood entries for BUM traffic. After setup, hosts on the overlay communicate as if directly connected on a LAN. Adding a third host is simply adding another VTEP entry on all existing hosts.
