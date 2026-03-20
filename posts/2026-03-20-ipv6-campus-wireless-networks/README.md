# How to Configure IPv6 for Campus Wireless Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Campus Network, Wireless LAN, eduroam, DHCPv6, Prefix Delegation, Enterprise Wi-Fi

Description: Deploy IPv6 across campus wireless networks with hierarchical prefix delegation, per-building or per-SSID /64 allocations, eduroam IPv6 support, and centralized monitoring.

---

Campus wireless networks serve thousands of clients across multiple buildings and SSIDs. IPv6 deployment requires hierarchical prefix delegation from the campus ISP, per-building /48 or /56 sub-prefixes, and proper integration with eduroam federation for guest research networking.

## Campus IPv6 Addressing Plan

```
Campus IPv6 Addressing Hierarchy:

ISP allocates: 2001:db8:university::/48

Building A:    2001:db8:university:a00::/56  → /64 per SSID/VLAN
  Corp SSID:   2001:db8:university:a00::/64
  Student SSID: 2001:db8:university:a01::/64
  IoT SSID:    2001:db8:university:a02::/64
  Guest SSID:  2001:db8:university:a03::/64

Building B:    2001:db8:university:b00::/56
  Corp SSID:   2001:db8:university:b00::/64
  Student SSID: 2001:db8:university:b01::/64
  ...

Core Infrastructure: 2001:db8:university:ff00::/56
  Routers:     2001:db8:university:ff00::/64
  Switches:    2001:db8:university:ff01::/64
  APs:         2001:db8:university:ff02::/64
```

## Campus Router Prefix Delegation (Cisco IOS)

```
! Campus distribution router - prefix delegation to building switches

ipv6 unicast-routing
ipv6 dhcp pool BUILDING-A
 prefix-delegation pool BLDGA-POOL
 dns-server 2001:db8:university::dns
 domain-name example.edu

ipv6 local pool BLDGA-POOL 2001:db8:university:a00::/56 64

interface GigabitEthernet0/0.10
 description Building-A uplink
 encapsulation dot1Q 10
 ipv6 address 2001:db8:university:a00::1/64
 ipv6 nd ra-interval 30
 ipv6 dhcp server BUILDING-A
 ipv6 nd managed-config-flag

! RA for SLAAC
interface GigabitEthernet0/0.10
 ipv6 nd prefix 2001:db8:university:a00::/64
```

## eduroam IPv6 Support

```
eduroam IPv6 Architecture:
[Wi-Fi Client with eduroam credentials]
    → [Local AP/Controller]
        → [Local RADIUS server]
            → [eduroam RADIUS federation]
                → [Home institution RADIUS]

IPv6 RADIUS attributes for eduroam:
- Framed-IPv6-Address (attribute 168): Assigned IPv6 address
- Framed-IPv6-Prefix (attribute 97): Assigned prefix
- Framed-Interface-Id (attribute 96): Interface identifier

FreeRADIUS eduroam IPv6 configuration:
```

```bash
# /etc/freeradius/3.0/sites-enabled/eduroam

server eduroam {
    listen {
        type = auth
        ipaddr = 2001:db8::radius
        port = 1812
    }

    authorize {
        # Check realm for federation
        suffix
        eduroam_outer
    }

    post-auth {
        # Assign IPv6 prefix for eduroam clients
        update reply {
            Framed-IPv6-Prefix = "2001:db8:university:eduroam::/64"
        }
    }
}
```

## hostapd with IPv6 for Campus APs

```bash
# /etc/hostapd/corp-ssid.conf
interface=wlan0
driver=nl80211
ssid=UniversityCorp
country_code=US
hw_mode=a
channel=36
ieee80211n=1
ieee80211ac=1

# WPA2-Enterprise for campus
wpa=2
wpa_key_mgmt=WPA-EAP
ieee8021x=1
auth_server_addr=2001:db8::radius   # IPv6 RADIUS server
auth_server_port=1812
auth_server_shared_secret=radiussecret

# IPv6 is handled by the bridge interface, not hostapd directly
bridge=br-corp
```

## Hierarchical DHCPv6 for Campus

```bash
# /etc/dhcp/dhcpd6.conf - Campus DHCPv6

# DNS options for campus
option dhcp6.name-servers 2001:db8:university::dns;
option dhcp6.domain-search "example.edu" "student.example.edu";

# Corporate SSID
subnet6 2001:db8:university:a00::/64 {
    range6 2001:db8:university:a00::100 2001:db8:university:a00::ffff;
    option dhcp6.name-servers 2001:db8:university::dns;
    default-lease-time 86400;
}

# Student SSID
subnet6 2001:db8:university:a01::/64 {
    range6 2001:db8:university:a01::100 2001:db8:university:a01::ffff;
    option dhcp6.name-servers 2001:db8:university::dns;
    # Students get shorter leases
    default-lease-time 43200;
    max-lease-time 86400;
}

# Guest SSID with public DNS
subnet6 2001:db8:university:a03::/64 {
    range6 2001:db8:university:a03::100 2001:db8:university:a03::ffff;
    option dhcp6.name-servers 2606:4700:4700::1111;
    default-lease-time 3600;
}
```

## Monitor Campus IPv6 Clients

```python
#!/usr/bin/env python3
# campus_ipv6_monitor.py - Monitor IPv6 adoption per SSID/building

import subprocess
import re
from collections import defaultdict

def get_dhcp6_leases():
    """Parse DHCPv6 lease file."""
    leases = []
    try:
        with open('/var/lib/dhcpd/dhcpd6.leases') as f:
            content = f.read()
    except FileNotFoundError:
        return leases

    # Parse leases
    lease_blocks = re.findall(r'ia-na "(.*?)" \{(.*?)\}', content, re.DOTALL)
    for duid, block in lease_blocks:
        addr_match = re.search(r'iaaddr ([\da-f:]+)', block)
        if addr_match:
            leases.append({
                'duid': duid,
                'address': addr_match.group(1)
            })
    return leases

def categorize_by_prefix(leases):
    """Categorize leases by building/SSID prefix."""
    buildings = defaultdict(list)

    for lease in leases:
        addr = lease['address']
        # Extract /48 prefix for building classification
        parts = addr.split(':')
        if len(parts) >= 4:
            building = ':'.join(parts[:4])
            buildings[building].append(addr)

    return buildings

if __name__ == '__main__':
    leases = get_dhcp6_leases()
    print(f"Total DHCPv6 leases: {len(leases)}")
    buildings = categorize_by_prefix(leases)
    for bldg, addrs in sorted(buildings.items()):
        print(f"  {bldg}:: → {len(addrs)} clients")
```

Campus IPv6 wireless deployment benefits from hierarchical prefix delegation that mirrors the physical topology (campus → building → floor → SSID), enabling meaningful IPv6 address assignments that simplify network management, troubleshooting, and policy enforcement across large multi-building wireless deployments.
