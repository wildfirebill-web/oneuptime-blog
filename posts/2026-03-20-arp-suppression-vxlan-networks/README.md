# How to Handle ARP Suppression in VXLAN Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VXLAN, ARP Suppression, Overlay Network, Networking, Performance, SDN

Description: Configure ARP suppression in VXLAN networks to reduce BUM flooding by having VTEPs respond to ARP requests locally using cached IP-to-MAC mappings.

## Introduction

In VXLAN networks, ARP requests are flooded to all VTEPs as BUM (Broadcast, Unknown, Multicast) traffic. At scale with hundreds of VTEPs and VMs, this generates significant overhead. ARP suppression allows a VTEP to respond to ARP requests locally using a local IP-to-MAC cache, eliminating unnecessary flooding.

## Enable ARP Proxy on Linux VXLAN

Linux supports ARP proxy on VXLAN interfaces, which intercepts ARP requests and replies if the target IP is known:

```bash
# Create VXLAN with ARP proxy enabled
ip link add vxlan0 type vxlan \
    id 100 \
    dstport 4789 \
    local 10.0.0.1 \
    dev eth0 \
    proxy        # Enable ARP proxy/suppression
```

## Enable ARP Proxy on Existing VXLAN

```bash
# Enable ARP proxy on an existing VXLAN interface
ip link set vxlan0 type vxlan proxy on

# Verify
ip -d link show vxlan0 | grep proxy
```

## Populate the ARP Cache (Neighbor Table)

For ARP suppression to work, the VTEP must know the MAC-to-IP mappings. Add them manually or through a control plane:

```bash
# Add a static ARP entry for a remote VM
# This tells the VTEP to reply to ARP for 10.200.0.10 with MAC aa:bb:cc:dd:ee:11
# without flooding the request to all VTEPs
ip neighbor add 10.200.0.10 lladdr aa:bb:cc:dd:ee:11 dev vxlan0 nud permanent

# Also add the VXLAN FDB entry so the VTEP knows where to forward unicast frames
bridge fdb add aa:bb:cc:dd:ee:11 dev vxlan0 dst 10.0.0.2 permanent
```

## Verify ARP Suppression

```bash
# Check neighbor table entries
ip neigh show dev vxlan0

# Monitor ARP traffic
tcpdump -i br-vxlan arp -n

# With ARP suppression:
# - ARP requests for known IPs are answered locally
# - No ARP request is flooded via VXLAN
# - tcpdump should show fewer ARP broadcasts
```

## Control Plane Integration (BGP EVPN)

Production VXLAN deployments use BGP EVPN to distribute MAC-IP bindings automatically:

```
With BGP EVPN:
1. VM comes up with IP 10.200.0.10 and MAC aa:bb:cc:dd:ee:11
2. Local VTEP advertises MAC-IP binding via BGP EVPN (type-2 route)
3. All remote VTEPs receive the binding and populate their neighbor tables
4. ARP suppression works automatically without manual configuration
```

## Open vSwitch ARP Suppression

OVS has native ARP suppression support:

```bash
# Enable ARP responder in OVS
ovs-vsctl set bridge br-vxlan other_config:mac-table-size=50000

# OVS uses the learned FIB to answer ARP requests locally
```

## Impact on Network Traffic

```
Without ARP suppression:
- 100 VMs × ARP every 30s = high BUM flood rate
- Each ARP floods to all N VTEPs → O(N) packets per ARP

With ARP suppression:
- ARP answered locally by VTEP
- Only ARP for unknown IPs is flooded
- Reduces BUM traffic dramatically at scale
```

## Conclusion

ARP suppression dramatically reduces BUM flooding in large VXLAN deployments. Enable it with the `proxy` flag on the VXLAN interface and populate the neighbor table with static entries or via a control plane like BGP EVPN. Without ARP suppression, ARP floods scale linearly with the number of VMs and VTEPs. Production deployments should always use ARP suppression with an appropriate control plane for automatic MAC-IP distribution.
