# How to Plan IPv6 Addressing for SD-WAN

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, SD-WAN, Address Planning, Subnetting, Prefix Delegation, Network Design

Description: Plan a hierarchical IPv6 addressing scheme for SD-WAN deployments covering site prefixes, overlay tunnel addresses, management networks, and prefix delegation strategies.

---

IPv6 SD-WAN addressing requires a hierarchical plan that scales from a single ISP-assigned prefix to thousands of sites. A well-designed scheme enables summarization, policy-by-address, and easy troubleshooting by embedding site identity in the IPv6 address structure.

## IPv6 Prefix Allocation Strategy

```text
IPv6 SD-WAN Addressing Hierarchy:

ISP Allocation: 2001:db8:company::/48
  (65,536 /64 subnets available)

Breakdown:
  2001:db8:company:0000::/52   → Region 1 (4096 /64s)
  2001:db8:company:1000::/52   → Region 2 (4096 /64s)
  2001:db8:company:2000::/52   → Region 3 (4096 /64s)
  ...
  2001:db8:company:f000::/52   → Infrastructure/Mgmt

Region 1 breakdown:
  2001:db8:company:0000::/56  → Site 1 (256 /64s)
  2001:db8:company:0100::/56  → Site 2 (256 /64s)
  2001:db8:company:0200::/56  → Site 3 (256 /64s)
  ...

Site breakdown (/56 = 256 /64s):
  :xx00::/64  → LAN - Corporate users
  :xx01::/64  → LAN - VoIP
  :xx02::/64  → LAN - IoT/OT
  :xx03::/64  → LAN - Guest/DMZ
  :xxfe::/64  → WAN interfaces
  :xxff::/64  → Management
```

## Address Encoding Scheme

```text
Structured IPv6 Address Encoding:

2001 : 0db8 : aabb : ccdd : xxxx : xxxx : xxxx : xxxx
 |       |     |     |
 |       |   OUI   Dept   ← Site/Department encoding
 |    Company
Global prefix (from ISP)

Example encoding:
2001:db8:0001:0100::/64
           ||||  ||||
           ||||  ++++ = VLAN/subnet within site
           ++++ = Site ID (1-65535)

Site 1, VLAN 0 (Corp): 2001:db8:0001:0100::/64
Site 1, VLAN 1 (VoIP): 2001:db8:0001:0101::/64
Site 2, VLAN 0 (Corp): 2001:db8:0002:0100::/64
Site 255, VLAN 3 (Guest): 2001:db8:00ff:0103::/64
```

## Address Planning Spreadsheet Logic

```python
#!/usr/bin/env python3
# ipv6_sdwan_address_plan.py - Generate SD-WAN IPv6 address plan

import ipaddress
import json

BASE_PREFIX = "2001:db8:company::/48"
SITES = [
    {"id": 1, "name": "HQ-New-York", "region": 1},
    {"id": 2, "name": "Branch-London", "region": 2},
    {"id": 3, "name": "Branch-Tokyo", "region": 2},
    {"id": 100, "name": "DC-Primary", "region": 3},
]

VLANS = [
    {"id": 0, "name": "Corporate", "suffix": "00"},
    {"id": 1, "name": "VoIP", "suffix": "01"},
    {"id": 2, "name": "IoT", "suffix": "02"},
    {"id": 3, "name": "Guest", "suffix": "03"},
    {"id": 254, "name": "WAN", "suffix": "fe"},
    {"id": 255, "name": "Management", "suffix": "ff"},
]

def generate_site_prefix(base, site_id):
    """Generate /56 prefix for a site."""
    # Site ID encoded in 3rd hextet: 2001:db8:company:SSSx::/56
    # where SSS = site_id in hex
    network = ipaddress.IPv6Network(base)
    site_hex = f"{site_id:04x}"
    # Modify 3rd hextet
    prefix_str = f"2001:db8:company:{site_hex}00::/56"
    return ipaddress.IPv6Network(prefix_str, strict=False)

def generate_vlan_prefix(site_prefix, vlan_id):
    """Generate /64 for specific VLAN within a site."""
    # Subnet the /56 into /64s by VLAN ID
    subnets = list(site_prefix.subnets(new_prefix=64))
    return subnets[vlan_id] if vlan_id < len(subnets) else None

address_plan = {}
for site in SITES:
    site_prefix = generate_site_prefix(BASE_PREFIX, site["id"])
    site_data = {
        "site_name": site["name"],
        "region": site["region"],
        "site_prefix_56": str(site_prefix),
        "vlans": {}
    }

    for vlan in VLANS:
        vlan_prefix = generate_vlan_prefix(site_prefix, vlan["id"])
        if vlan_prefix:
            site_data["vlans"][vlan["name"]] = {
                "prefix_64": str(vlan_prefix),
                "gateway": str(list(vlan_prefix.hosts())[0]),
                "dhcpv6_start": str(list(vlan_prefix.hosts())[99]),
                "dhcpv6_end": str(list(vlan_prefix.hosts())[999])
            }

    address_plan[site["id"]] = site_data

# Print summary

for site_id, data in address_plan.items():
    print(f"\nSite {site_id}: {data['site_name']}")
    print(f"  Site prefix (/56): {data['site_prefix_56']}")
    for vlan_name, vlan_data in data["vlans"].items():
        print(f"  {vlan_name:15} /64: {vlan_data['prefix_64']}")
```

## SD-WAN Overlay Addressing

```text
SD-WAN Overlay Address Ranges:

Management/Overlay infrastructure:
  2001:db8:company:f000::/52

Controller/Orchestrator:
  2001:db8:company:f000::10/128  → vManage/Orchestrator
  2001:db8:company:f000::11/128  → vBond/Secondary
  2001:db8:company:f000::12/128  → vSmart

Tunnel Endpoints (loopbacks):
  2001:db8:company:f001:0001::/80 → Site 1 SD-WAN edge loopback
  2001:db8:company:f001:0002::/80 → Site 2 SD-WAN edge loopback
  ...
  (encoded: f001:SSSS:: where SSSS = site ID)

NTP, DNS, RADIUS for SD-WAN:
  2001:db8:company:f002::/64
```

## Route Summarization by Region

```text
BGP Route Summarization:

Region 1 (Americas): Summarize as 2001:db8:company:0000::/50
Region 2 (EMEA):     Summarize as 2001:db8:company:1000::/50
Region 3 (APAC):     Summarize as 2001:db8:company:2000::/50
Infrastructure:      Summarize as 2001:db8:company:f000::/52

Regional hubs advertise summaries upstream,
branches advertise /56 site prefixes to regional hub.

BGP configuration:
  aggregate-address 2001:db8:company:0000::/50 summary-only
  (Cisco IOS - aggregate and suppress more specifics)
```

A successful IPv6 SD-WAN addressing plan encodes site identity, region, and VLAN type directly into the address structure, enabling meaningful route summarization at regional hub routers and simplifying firewall policies that can match `2001:db8:company:0001::/56` to identify all traffic from a specific site.
