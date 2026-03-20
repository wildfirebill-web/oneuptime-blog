# How to Configure IPv6 with OVS (Open vSwitch)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Open vSwitch, OVS, SDN, Virtual Networking, OpenFlow

Description: Configure Open vSwitch (OVS) for IPv6 traffic with VLAN tagging, flow rules for IPv6, tunnel endpoints over IPv6, and integration with OpenStack and Kubernetes networking.

## Introduction

Open vSwitch (OVS) is a software-defined networking virtual switch that supports IPv6 traffic forwarding, VXLAN/Geneve tunnels over IPv6, and OpenFlow rules for IPv6 packet matching. OVS operates at L2 and is IPv6-transparent for most bridging use cases, but requires specific configuration for IPv6-aware flow rules, tunnels with IPv6 endpoints, and QoS policies.

## Basic OVS Bridge with IPv6

```bash
# Install Open vSwitch
apt-get install -y openvswitch-switch

# Create an OVS bridge
ovs-vsctl add-br ovs-br0

# Add a physical interface to the bridge
ovs-vsctl add-port ovs-br0 eth0

# Assign IPv6 address to the bridge (host management)
ip -6 addr add 2001:db8::10/64 dev ovs-br0
ip -6 route add default via 2001:db8::1 dev ovs-br0

# Verify
ovs-vsctl show
ip -6 addr show ovs-br0
```

## VLAN-Aware OVS for IPv6

```bash
# Create VLAN-tagged ports on OVS bridge
# IPv6 traffic passes through VLANs transparently

# Add access port (VLAN 100 for VM)
ovs-vsctl add-port ovs-br0 vnet0 tag=100

# Add trunk port (multiple VLANs, for uplink)
ovs-vsctl add-port ovs-br0 eth1 \
    vlan_mode=trunk \
    trunks=100,200,300

# Patch port between two OVS bridges
ovs-vsctl add-port ovs-br0 patch-to-br1 -- \
    set interface patch-to-br1 type=patch options:peer=patch-to-br0

ovs-vsctl add-port ovs-br1 patch-to-br0 -- \
    set interface patch-to-br0 type=patch options:peer=patch-to-br1

# IPv6 traffic flows through patch ports without special configuration
```

## VXLAN Tunnel over IPv6 (OVS)

```bash
# Create VXLAN tunnel with IPv6 endpoints
# Both tunnel endpoints must have IPv6 addresses

# On host1 (2001:db8::10): create VXLAN to host2 (2001:db8::11)
ovs-vsctl add-port ovs-br0 vxlan0 -- \
    set interface vxlan0 type=vxlan \
    options:remote_ip=2001:db8::11 \
    options:local_ip=2001:db8::10 \
    options:key=1000               # VNI (VXLAN Network Identifier)

# On host2 (2001:db8::11): create VXLAN to host1
ovs-vsctl add-port ovs-br0 vxlan0 -- \
    set interface vxlan0 type=vxlan \
    options:remote_ip=2001:db8::10 \
    options:local_ip=2001:db8::11 \
    options:key=1000

# Verify tunnel is established
ovs-vsctl show | grep -A5 vxlan0

# Test: VMs on both hosts can reach each other over the IPv6 tunnel
```

## Geneve Tunnel over IPv6

```bash
# Geneve tunnel with IPv6 underlay (preferred over VXLAN for extensibility)
ovs-vsctl add-port ovs-br0 geneve0 -- \
    set interface geneve0 type=geneve \
    options:remote_ip=2001:db8::11 \
    options:local_ip=2001:db8::10 \
    options:key=flow               # VNI from OpenFlow

# Check Geneve tunnel status
ovs-vsctl get interface geneve0 statistics
```

## OpenFlow Rules for IPv6 Traffic

```bash
# Add OpenFlow rules to match and handle IPv6 traffic

# Allow all IPv6 traffic through bridge
ovs-ofctl add-flow ovs-br0 \
    "priority=100,ipv6,actions=normal"

# Match ICMPv6 NDP Neighbor Solicitation (type=135)
ovs-ofctl add-flow ovs-br0 \
    "priority=200,icmp6,icmpv6_type=135,actions=normal"

# Match ICMPv6 Router Advertisement (type=134)
ovs-ofctl add-flow ovs-br0 \
    "priority=200,icmp6,icmpv6_type=134,actions=normal"

# Drop IPv6 from untrusted source prefix
ovs-ofctl add-flow ovs-br0 \
    "priority=50,ipv6,ipv6_src=2001:db8:bad::/48,actions=drop"

# Send IPv6 traffic to specific output port
ovs-ofctl add-flow ovs-br0 \
    "priority=100,ipv6,ipv6_dst=2001:db8::100,actions=output:2"

# Show all flows
ovs-ofctl dump-flows ovs-br0
```

## OVS with OpenStack Neutron

```bash
# OpenStack uses OVS via the neutron-openvswitch-agent
# Check OVS configuration created by Neutron

# List OVS bridges created by Neutron
ovs-vsctl list-br
# Typical: br-int (integration), br-tun (tunnels), br-ex (external)

# Check tunnel ports (VXLAN over IPv6 if configured)
ovs-vsctl list-ports br-tun

# View Neutron's OpenFlow rules for IPv6
ovs-ofctl dump-flows br-int | grep ipv6

# /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
# For IPv6 VXLAN tunnels
local_ip = 2001:db8::10    # OVS tunnel endpoint IPv6 address
tunnel_types = vxlan
```

## OVS with Kubernetes (OVN)

```bash
# OVN (Open Virtual Network) uses OVS for Kubernetes networking
# Check OVN configuration for IPv6

# List logical routers (handles IPv6 routing)
ovn-nbctl lr-list

# Show logical switch with IPv6 subnet
ovn-nbctl ls-list
ovn-nbctl lsp-list my-logical-switch

# Check OVN's IPv6 routes
ovn-nbctl lr-route-list my-logical-router

# OVS datapath shows IPv6 megaflows
ovs-dpctl dump-flows | grep "ipv6"
```

## Verify OVS IPv6 Operation

```bash
# Monitor IPv6 traffic through OVS
ovs-ofctl monitor ovs-br0 "watch:ipv6"

# Packet capture on OVS port
tcpdump -i ovs-br0 -n ip6

# Check port statistics
ovs-ofctl dump-ports ovs-br0

# Debug IPv6 flow lookup
ovs-appctl ofproto/trace ovs-br0 "ipv6,in_port=1,ipv6_src=2001:db8::1"
```

## Conclusion

Open vSwitch handles IPv6 traffic transparently at L2 for standard bridging and VLAN scenarios. OpenFlow rules match IPv6 with the `ipv6` protocol specifier and specific `icmp6` type matches for NDP traffic. VXLAN and Geneve tunnels support IPv6 underlay addresses via `options:remote_ip` and `options:local_ip` with IPv6 addresses. OpenStack Neutron configures OVS automatically for IPv6 when tunnel endpoints use IPv6 addresses in `local_ip`. For explicit flow control of IPv6, use `ovs-ofctl add-flow` with `ipv6` or `icmp6` matchers to implement security and routing policies within the virtual switch.
