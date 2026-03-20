# How to Configure IPv6 on HPE/Aruba Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, HPE, Aruba, Switch, ArubaOS, Networking

Description: Configure IPv6 on HPE/Aruba switches including VLAN SVIs, OSPFv3, and IPv6 access control lists for enterprise campus and datacenter deployments.

## Introduction

HPE/Aruba switches running ArubaOS-CX (modern) or ProVision/AdvancedTraffic (legacy) support full IPv6. This guide focuses on ArubaOS-CX (used in Aruba CX 6xxx, 8xxx, and 10xxx series) as it represents the current platform.

## ArubaOS-CX IPv6 Configuration

### Step 1: Enable IPv6 Routing

```bash
# Enable IPv6 unicast routing globally
switch# configure terminal
switch(config)# ipv6 unicast-routing
```

### Step 2: Configure IPv6 on a VLAN Interface (SVI)

```bash
# Configure VLAN 100 with IPv6
switch(config)# interface vlan 100
switch(config-if-vlan)# ipv6 address 2001:db8:1:100::1/64
switch(config-if-vlan)# ipv6 enable
switch(config-if-vlan)# no shutdown
```

### Step 3: Configure IPv6 on a Routed Port

```bash
# Convert a port to routed mode and assign IPv6
switch(config)# interface 1/1/1
switch(config-if)# no switchport
switch(config-if)# ipv6 address 2001:db8:0:1::1/64
switch(config-if)# no shutdown
```

### Step 4: Configure Router Advertisements

```bash
# Enable RA on a VLAN interface
switch(config)# interface vlan 100
switch(config-if-vlan)# ipv6 nd ra interval 100
switch(config-if-vlan)# ipv6 nd ra lifetime 1800

# Configure prefix advertisement
switch(config-if-vlan)# ipv6 nd prefix 2001:db8:1:100::/64 valid-lifetime 86400 preferred-lifetime 14400
switch(config-if-vlan)# ipv6 nd ra m-flag disable
switch(config-if-vlan)# ipv6 nd ra o-flag disable

# Advertise DNS via RDNSS
switch(config-if-vlan)# ipv6 nd ra dns-server 2001:db8:1:100::53 lifetime 600
```

### Step 5: Configure Static Routes

```bash
# Static route
switch(config)# ipv6 route 2001:db8:2::/48 2001:db8:0:1::2

# Default route
switch(config)# ipv6 route ::/0 2001:db8:isp::1
```

### Step 6: Configure OSPFv3

```bash
# Configure OSPFv3
switch(config)# router ospfv3 1
switch(config-ospfv3-1)# router-id 1.1.1.1

# Enable on interfaces
switch(config)# interface vlan 100
switch(config-if-vlan)# ipv6 ospfv3 1 area 0

switch(config)# interface 1/1/1
switch(config-if)# ipv6 ospfv3 1 area 0
```

### Step 7: Configure IPv6 ACL

```bash
# Create an IPv6 access control list
switch(config)# ipv6 access-list IPV6-FILTER

# Permit ICMPv6 (required for Neighbor Discovery)
switch(config-ipv6-acl)# permit icmpv6 any any

# Permit established TCP
switch(config-ipv6-acl)# permit tcp any any established

# Deny everything else
switch(config-ipv6-acl)# deny ipv6 any any

# Apply to interface
switch(config)# interface 1/1/1
switch(config-if)# ipv6 access-group IPV6-FILTER in
```

## Legacy HPE ProVision/AdvancedTraffic IPv6

For older HPE switches (5400, 3800 series):

```bash
# Enable IPv6 routing
switch# ipv6 routing

# Configure a VLAN with IPv6
switch# vlan 100
switch(vlan-100)# ipv6 address 2001:db8:1:100::1/64

# Add a static route
switch# ipv6 route ::/0 2001:db8:isp::1
```

## Verification Commands (ArubaOS-CX)

```bash
# Show IPv6 interface addresses
switch# show ipv6 interface brief

# Show IPv6 routing table
switch# show ipv6 route

# Show OSPFv3 neighbors
switch# show ipv6 ospf neighbor

# Show IPv6 neighbor discovery cache
switch# show ipv6 neighbors

# Show RA configuration
switch# show ipv6 nd

# Ping test
switch# ping6 2606:4700:4700::1111
```

## Conclusion

HPE/Aruba ArubaOS-CX switches provide comprehensive IPv6 support through a CLI structure that closely resembles Cisco IOS. Key differences include the `ipv6 enable` command for per-interface IPv6 activation and the `ipv6 ospfv3` syntax for OSPFv3. The switch can serve as a default gateway for IPv6 VLAN segments with integrated RA support, eliminating the need for a separate radvd server.
