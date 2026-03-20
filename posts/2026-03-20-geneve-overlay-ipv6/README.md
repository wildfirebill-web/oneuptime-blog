# How to Configure Geneve Overlay with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GENEVE, IPv6, Overlay, Linux, OVN, Networking

Description: Configure Geneve overlay tunnels over IPv6 on Linux including manual setup, OVN integration, and performance comparison with VXLAN.

## Geneve vs VXLAN

Geneve (Generic Network Virtualization Encapsulation, RFC 8926) is a next-generation encapsulation protocol that supersedes VXLAN and NVGRE:

| Feature | VXLAN | Geneve |
|---|---|---|
| VNI bits | 24 | 24 |
| Header extensions | No | Yes (TLV options) |
| Metadata support | Limited | Rich (OVN metadata) |
| UDP port | 4789 | 6081 |
| IPv6 underlay | Yes | Yes |

## Creating Geneve Tunnel on Linux

```bash
# Load the Geneve kernel module

modprobe geneve

# Create Geneve tunnel over IPv6
# VNI 100, local VTEP at 2001:db8:1::1
ip link add geneve100 type geneve \
    id 100 \
    remote 2001:db8:2::1 \
    dstport 6081

ip link set geneve100 up
ip addr add 10.0.0.1/30 dev geneve100

echo "Geneve tunnel created"

# Show tunnel details
ip -d link show geneve100
```

## Point-to-Point Geneve with IPv6

For a simple point-to-point overlay between two hosts:

```bash
#!/bin/bash
# Host A: 2001:db8:1::1
# Host B: 2001:db8:2::1

# On Host A
create_geneve_p2p() {
    local LOCAL=$1
    local REMOTE=$2
    local VNI=$3
    local OVERLAY_ADDR=$4

    ip link add geneve${VNI} type geneve \
        id ${VNI} \
        remote ${REMOTE} \
        dstport 6081

    ip link set geneve${VNI} up
    ip addr add ${OVERLAY_ADDR} dev geneve${VNI}
    ip link set geneve${VNI} mtu 1442  # 1500 - 58 bytes Geneve/IPv6 overhead

    echo "Geneve VNI ${VNI}: ${LOCAL} → ${REMOTE}"
}

# Create tunnels on Host A
create_geneve_p2p "2001:db8:1::1" "2001:db8:2::1" 100 "10.0.0.1/30"
create_geneve_p2p "2001:db8:1::1" "2001:db8:3::1" 200 "10.0.1.1/30"
```

## OVN with IPv6 Underlay (Geneve)

Open Virtual Network (OVN) uses Geneve by default and supports IPv6 tunnels:

```bash
# Configure OVS/OVN to use IPv6 for tunnel endpoints
# Set the local tunnel IP on this host
ovs-vsctl set Open_vSwitch . \
    other_config:ovn-encap-type=geneve \
    other_config:ovn-encap-ip=2001:db8:1::1

# Point to OVN Southbound database
ovs-vsctl set Open_vSwitch . \
    external_ids:ovn-remote=ssl:[2001:db8::db]:6642

# Show current encap config
ovs-vsctl get Open_vSwitch . other_config

# Verify tunnel ports
ovs-vsctl show | grep -A 3 tun
```

## Geneve Overhead Calculation

```text
Geneve over IPv6 encapsulation overhead:

  Outer Ethernet:  14 bytes (L2 frame)
  Outer IPv6:      40 bytes
  Outer UDP:        8 bytes
  Geneve header:    8 bytes (base, no options)
  ─────────────────────────
  Total overhead:  70 bytes (no options)

With TLV options (e.g., OVN metadata):
  + 4-8 bytes per option
  Typical total: 78-86 bytes

VXLAN over IPv6: 62 bytes (8 less than Geneve base)
```

```bash
# Calculate effective MTU for Geneve over IPv6
PHYS_MTU=1500
GENEVE_OVERHEAD=70

EFFECTIVE_MTU=$((PHYS_MTU - GENEVE_OVERHEAD))
echo "Effective MTU for Geneve/IPv6: ${EFFECTIVE_MTU}"  # 1430

# Set MTU on Geneve interface
ip link set geneve100 mtu ${EFFECTIVE_MTU}
```

## Monitoring Geneve Tunnels

```bash
# Capture Geneve traffic (UDP port 6081)
tcpdump -i eth0 -n -v 'ip6 and udp port 6081'

# Check tunnel statistics
ip -s link show geneve100

# Test tunnel connectivity
ping -I geneve100 10.0.0.2

# Verify encapsulation with traceroute
traceroute -s 2001:db8:1::1 2001:db8:2::1

# Check OVN tunnel status
ovs-vsctl show | grep -E 'Port|options'
```

## Conclusion

Geneve over IPv6 provides a flexible overlay with extensible metadata support via TLV options - critical for OVN's per-flow context. The Linux kernel supports Geneve natively with `ip link add ... type geneve`. UDP port 6081 is the standard Geneve port. Overhead is 70 bytes for base Geneve over IPv6, slightly more than VXLAN's 62 bytes, but the TLV extensions enable richer network virtualization semantics. OVN deployments automatically use IPv6 underlay when `ovn-encap-ip` is set to an IPv6 address.
