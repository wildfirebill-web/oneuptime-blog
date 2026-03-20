# How to Plan IPv6 Addressing for Data Center Fabrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Data Center, Addressing, Fabric, BGP, OSPF, Infrastructure

Description: Design a scalable IPv6 addressing scheme for data center fabrics including spine/leaf addressing, loopback assignments, management networks, and server/workload IPv6 prefixes.

---

Data center IPv6 addressing requires a hierarchical scheme that supports routing protocols (BGP, OSPF), management access, workload addressing, and future growth. Using a structured addressing plan that encodes role (spine/leaf/server) and location (row/rack) enables meaningful route summarization and easy troubleshooting.

## Data Center IPv6 Addressing Framework

```text
Data Center IPv6 Addressing Block:
/48 per DC from corporate allocation: 2001:db8:dc1::/48

Infrastructure:
  2001:db8:dc1:0000::/52   → Fabric (spine/leaf interconnects)
  2001:db8:dc1:1000::/52   → Management (OOB, IPMI, console)
  2001:db8:dc1:2000::/52   → Servers / Workloads
  2001:db8:dc1:f000::/52   → Reserved

Fabric /52 breakdown:
  2001:db8:dc1:0100::/56   → Spine Loopbacks
  2001:db8:dc1:0200::/56   → Leaf Loopbacks
  2001:db8:dc1:0300::/56   → Spine-Leaf P2P Links (/126 or /127 each)

Server /52 breakdown (by pod/rack):
  2001:db8:dc1:2000::/56   → Pod 1 (256 /64s per pod)
  2001:db8:dc1:2100::/56   → Pod 2
  2001:db8:dc1:2200::/56   → Pod 3
  ...
  Each /64 = one rack or one VLAN segment
```

## Spine and Leaf Loopback Addressing

```text
Spine Loopback Addressing (/128):
  Spine-1:  2001:db8:dc1:0100::1/128
  Spine-2:  2001:db8:dc1:0100::2/128
  Spine-3:  2001:db8:dc1:0100::3/128
  Spine-4:  2001:db8:dc1:0100::4/128

Leaf Loopback Addressing (/128):
  Leaf-101: 2001:db8:dc1:0200::101/128  (row 1, rack 01)
  Leaf-102: 2001:db8:dc1:0200::102/128
  Leaf-201: 2001:db8:dc1:0200::201/128  (row 2, rack 01)
  ...
  Encoding: 0200::RRrr where RR=row, rr=rack

P2P Link Addressing (/126 or /127):
  Spine-1 <-> Leaf-101: 2001:db8:dc1:0300:1:101::/127
    Spine-1 port: ...::0
    Leaf-101 port: ...::1
  Spine-1 <-> Leaf-102: 2001:db8:dc1:0300:1:102::/127
    Spine-1 port: ...::0
    Leaf-102 port: ...::1
```

## BGP Underlay IPv6 Configuration (Cisco Nexus)

```text
! Leaf-101 - Cisco Nexus BGP configuration for IPv6 fabric

feature bgp
feature interface-vlan
ipv6 unicast-routing

! Loopback interface
interface loopback0
  ipv6 address 2001:db8:dc1:0200::101/128

! Spine-1 facing interface
interface Ethernet1/1
  description Spine-1
  no switchport
  ipv6 address 2001:db8:dc1:0300:1:101::1/127
  no shutdown

! BGP configuration (eBGP with Spine)
router bgp 65101
  router-id 10.0.1.101
  address-family ipv6 unicast
    network 2001:db8:dc1:0200::101/128   ! Loopback

  neighbor 2001:db8:dc1:0300:1:101::0   ! Spine-1
    remote-as 65001
    address-family ipv6 unicast
      activate
```

## Server Workload IPv6 Addressing

```text
Server Workload Addressing (/64 per VLAN):

Pod 1, Rack 1:
  VLAN 100 (App Tier): 2001:db8:dc1:2001::/64
    Servers: ...::server-mac-derived (SLAAC)
    Or DHCPv6: ...::100 to ...::200

  VLAN 200 (DB Tier): 2001:db8:dc1:2002::/64

Pod 1, Rack 2:
  VLAN 100: 2001:db8:dc1:2011::/64
  VLAN 200: 2001:db8:dc1:2012::/64

Container Workloads (Kubernetes):
  Each node gets a /64: 2001:db8:dc1:3NNN::/64
    Where NNN = node number
  Each pod gets a /128 from the node's /64
```

## BGP Spine Configuration

```text
! Spine-1 - Aggregates leaf routes

router bgp 65001
  router-id 10.0.0.1

  address-family ipv6 unicast
    ! Aggregate leaf loopback block
    aggregate-address 2001:db8:dc1:0200::/56 summary-only

  ! Dynamic BGP neighbors for all leaf connections
  neighbor 2001:db8:dc1:0300:1::/80   ! All /127 from Spine-1 to leaves
    remote-as range 65100 65199
    address-family ipv6 unicast
      activate
      route-map LEAF-IN in
      route-map SPINE-OUT out

! Only accept leaf loopbacks and workload prefixes
ip prefix-list LEAF-ALLOWED seq 10 permit 2001:db8:dc1:0200::/56 le 128
ip prefix-list LEAF-ALLOWED seq 20 permit 2001:db8:dc1:2000::/52 le 64
```

## IPv6 Addressing Documentation Template

```python
#!/usr/bin/env python3
# dc_ipv6_inventory.py - Generate DC IPv6 inventory

import ipaddress

DC_BASE = "2001:db8:dc1::/48"
SPINE_LOOPBACKS = "2001:db8:dc1:0100::/56"
LEAF_LOOPBACKS = "2001:db8:dc1:0200::/56"

spines = {
    "Spine-1": "2001:db8:dc1:0100::1/128",
    "Spine-2": "2001:db8:dc1:0100::2/128",
    "Spine-3": "2001:db8:dc1:0100::3/128",
    "Spine-4": "2001:db8:dc1:0100::4/128",
}

# Generate leaf loopbacks

leaves = {}
for rack in range(1, 48 + 1):  # 48 leaf switches
    name = f"Leaf-{rack:03d}"
    addr = f"2001:db8:dc1:0200::{rack:x}/128"
    leaves[name] = addr

print("=== DC IPv6 Spine Loopbacks ===")
for name, addr in spines.items():
    print(f"  {name}: {addr}")

print("\n=== DC IPv6 Leaf Loopbacks ===")
for name, addr in list(leaves.items())[:10]:
    print(f"  {name}: {addr}")
print(f"  ... ({len(leaves)} total)")
```

Data center fabric IPv6 addressing works best when it encodes switch role and location into the address structure, uses /127 or /126 point-to-point links between spine and leaf nodes, assigns /128 loopbacks for router IDs and iBGP peers, and allocates /64 server subnets from a separate block that enables route aggregation at pod boundaries.
