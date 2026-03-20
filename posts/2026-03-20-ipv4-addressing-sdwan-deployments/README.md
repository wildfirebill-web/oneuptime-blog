# How to Design IPv4 Addressing for SD-WAN Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, SD-WAN, Network Design, VPN, Cisco Viptela, VMware VeloCloud

Description: Design an IPv4 addressing plan for SD-WAN overlays covering transport (underlay), overlay tunnels, loopback IDs, service-side subnets, and management networks.

## Introduction

SD-WAN creates a virtual overlay over multiple transport links (MPLS, broadband, LTE). IPv4 addressing must cover both the underlay (physical transport) and overlay (virtual tunnel) networks while avoiding conflicts across all sites.

## SD-WAN Addressing Layers

```
Layer            Address Space       Purpose
───────────────────────────────────────────────────────
Underlay ISP     203.0.113.0/30      WAN uplinks (public/private)
Overlay Tunnel   10.255.0.0/16       SD-WAN fabric IPs
Loopback/System  10.254.0.0/16       Router system IDs
Service-Side     10.SITE.VLAN.0/24   LAN segments per site
Management OOB   10.253.0.0/16       Device management
```

## Site Numbering Scheme

```
Site ID: 3-digit number (001–999)
  001  HQ
  002  Branch Chicago
  003  Branch New York
  010  Data Center 1
  020  Data Center 2

Per-site address:
  Service LAN:  10.<site_id>.<vlan>.0/24
    Site 001:   10.1.10.0/24   (VLAN 10 — corporate)
                10.1.20.0/24   (VLAN 20 — guest)
    Site 002:   10.2.10.0/24
                10.2.20.0/24

Overlay loopback:
  10.254.<site_id>.1/32
    HQ (001):   10.254.1.1/32
    Chicago:    10.254.2.1/32
```

## Python Site Address Generator

```python
import ipaddress

SITES = {
    1:  "HQ",
    2:  "Chicago",
    3:  "New York",
    10: "DC1",
}

VLANS = {
    10: "Corporate",
    20: "Guest",
    30: "VoIP",
    99: "Mgmt",
}

for site_id, site_name in SITES.items():
    print(f"\nSite {site_id:03d} — {site_name}")
    loopback = f"10.254.{site_id}.1/32"
    print(f"  Loopback: {loopback}")
    for vlan, vlan_name in VLANS.items():
        subnet = f"10.{site_id}.{vlan}.0/24"
        net = ipaddress.IPv4Network(subnet)
        print(f"  VLAN {vlan} ({vlan_name:<10}): {subnet}  "
              f"GW: {list(net.hosts())[0]}")
```

## Cisco Viptela — Interface Configuration

```
! vEdge router at Site 002 (Chicago)
! Transport (underlay)
vpn 0
  interface ge0/0
    ip address 203.0.113.2/30
    tunnel-interface
      color biz-internet
      allow-service all

! Service side (LAN)
vpn 1
  name Service
  interface ge0/1
    ip address 10.2.10.1/24
  ip route 0.0.0.0/0 vpn 0
```

## Avoid Overlap Checklist

```
Before deployment, verify:
[ ] No site LAN prefix overlaps another site
[ ] Overlay tunnel space (10.255.x.x) doesn't collide with LAN
[ ] Loopback IDs are unique across all vEdges
[ ] AWS/Azure VNet CIDRs are excluded from site range
[ ] Remote-access VPN pool (if any) is separate
```

## Summarization at Hub Sites

```
HQ aggregates:
  10.1.0.0/16   — All HQ subnets
  Summary to MPLS/internet: 10.0.0.0/8  (entire SD-WAN)

Regional hub:
  10.1.0.0/20  — Sites 001–015 (if /24 per site per region)
```

## Conclusion

SD-WAN addressing requires coordinating underlay, overlay, LAN service, and management layers. Use a site-ID-embedded scheme (`10.<site>.<vlan>.0/24`) for automatic uniqueness and easy summarization. Document all allocations centrally before onboarding the first edge device.
