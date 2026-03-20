# How to Plan IPv6 Address Blocks for Data Center Racks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Address Planning, Data Center, Subnetting, IPAM

Description: Plan IPv6 address blocks for data center racks using a hierarchical allocation scheme that scales from rack to campus level.

## Why Rack-Level Planning Matters

Good IPv6 address planning at the rack level enables route summarization, simplifies ACL management, and makes troubleshooting easier. Each rack should have a predictable prefix derived from a hierarchical allocation.

## Recommended Hierarchy

```
/32  - Organization total
  /40  - Data Center site
    /48  - Pod or row
      /56  - Rack
        /64  - Individual VLAN/subnet within rack
```

This gives each rack a /56, providing 256 individual /64 subnets — enough for multiple VLANs per rack.

## Allocation Example

```
Site: DC-EAST     = 2001:db8:01::/40
Pod A             = 2001:db8:01:00::/48
  Row 1           = 2001:db8:01:00::/52
    Rack 1        = 2001:db8:01:00::/56
      VLAN 100    = 2001:db8:01:00:0100::/64   (servers)
      VLAN 200    = 2001:db8:01:00:0200::/64   (storage)
      VLAN 300    = 2001:db8:01:00:0300::/64   (IPMI/BMC)
    Rack 2        = 2001:db8:01:00:0100::/56
    Rack 3        = 2001:db8:01:00:0200::/56
```

## IPAM Spreadsheet Template

Track allocations in a structured IPAM tool or at minimum a spreadsheet:

| Site    | Pod | Row | Rack | VLAN | Prefix                  | Purpose   |
|---------|-----|-----|------|------|-------------------------|-----------|
| DC-EAST | A   | 1   | 1    | 100  | 2001:db8:1:100::/64     | Servers   |
| DC-EAST | A   | 1   | 1    | 200  | 2001:db8:1:200::/64     | Storage   |
| DC-EAST | A   | 1   | 2    | 100  | 2001:db8:1:1100::/64    | Servers   |

## Automating Allocation with Python

Use Python's `ipaddress` module to generate rack allocations programmatically:

```python
import ipaddress

# Start with a /48 for a pod
pod_prefix = ipaddress.IPv6Network("2001:db8:1::/48")

# Generate /56 prefixes for each rack (subnets of /48)
rack_prefixes = list(pod_prefix.subnets(new_prefix=56))

for i, rack in enumerate(rack_prefixes[:5]):
    print(f"Rack {i+1}: {rack}")
    # Each rack's /56 can be further divided into /64s
    vlans = list(rack.subnets(new_prefix=64))
    for j, vlan in enumerate(vlans[:3]):
        print(f"  VLAN {(j+1)*100}: {vlan}")
```

## Summarization at the ToR Switch

Ensure each ToR (Top-of-Rack) switch advertises only the rack's /56 summary route upstream, not individual /64s:

```
# FRR - advertise rack summary only
router bgp 65001
 address-family ipv6
  network 2001:db8:1::/56
  no network 2001:db8:1::/64
```

## Avoiding Common Mistakes

- Do not assign /128s to individual servers manually — use SLAAC or DHCPv6 within the /64.
- Reserve a /64 per rack for OOB management (IPMI/iDRAC/iLO).
- Document allocations in an IPAM system before deployment.

## Conclusion

A hierarchical IPv6 addressing plan for data center racks ensures scalability and manageability. By assigning /56 blocks per rack and using /64 subnets for individual VLANs, you can summarize routes efficiently and write clean ACLs that match entire racks or pods.
