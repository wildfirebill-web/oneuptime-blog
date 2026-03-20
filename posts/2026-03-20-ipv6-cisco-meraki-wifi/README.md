# How to Configure IPv6 on Cisco Meraki Wi-Fi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Cisco Meraki, Wi-Fi, Cloud Managed, SLAAC, DHCPv6, Wireless

Description: Enable and configure IPv6 on Cisco Meraki cloud-managed Wi-Fi networks including SLAAC, DHCPv6, prefix configuration, and client IPv6 address assignment through the Meraki Dashboard.

---

Cisco Meraki is a cloud-managed networking platform. IPv6 is configured through the Meraki Dashboard for MX security appliances and MR access points. Meraki supports dual-stack, SLAAC, and DHCPv6 for wireless clients.

## Meraki Dashboard IPv6 Configuration

```
Meraki Dashboard: Security & SD-WAN > Addressing & VLANs

For each VLAN/subnet:
1. Enable IPv6 on the interface
2. Set IPv6 prefix (static) or use DHCPv6-PD from upstream

Path: Network-wide > Configure > General > IPv6

Toggle: Enable IPv6
Mode options:
  - SLAAC (Stateless Address Autoconfiguration)
  - DHCPv6 Stateless (RA + DHCPv6 for DNS/options only)
  - DHCPv6 Stateful (full DHCPv6 address assignment)
```

## Meraki MX IPv6 Addressing

```
Meraki Dashboard > Security & SD-WAN > Appliance Status

WAN IPv6:
- Configure WAN IPv6 via Dashboard > Security & SD-WAN > WAN
- For ISP with DHCPv6-PD:
  Interface: WAN1
  IPv6 Assignment: Via prefix delegation
  Delegation: /48 or /56 from ISP

LAN IPv6:
Dashboard > Security & SD-WAN > Addressing & VLANs

VLAN 1 (Default):
  IPv6 prefix: 2001:db8:1::/64
  RA: Enabled
  DHCPv6: Stateless or Stateful
```

## Verify Meraki IPv6 via Dashboard

```
Dashboard > Network-wide > Clients

Filter by: IPv6 address
Shows all wireless clients with their IPv6 addresses

Dashboard > Security & SD-WAN > Event Log

Filter: IPv6
Shows DHCPv6 and RA events

Dashboard > Tools > Ping

Run ping6 from MX to verify upstream IPv6 connectivity
```

## Meraki API - Check IPv6 Client Addressing

```python
#!/usr/bin/env python3
# meraki_ipv6_clients.py - Query IPv6 clients via Meraki Dashboard API

import requests
import json

API_KEY = "your-meraki-api-key"
ORG_ID = "your-org-id"
NETWORK_ID = "your-network-id"

BASE_URL = "https://api.meraki.com/api/v1"

headers = {
    "X-Cisco-Meraki-API-Key": API_KEY,
    "Content-Type": "application/json"
}

def get_clients_with_ipv6(network_id):
    """Get all clients with IPv6 addresses."""
    url = f"{BASE_URL}/networks/{network_id}/clients"
    params = {"timespan": 3600, "perPage": 200}

    response = requests.get(url, headers=headers, params=params)
    clients = response.json()

    ipv6_clients = []
    for client in clients:
        if client.get('ip6') or client.get('ip6Local'):
            ipv6_clients.append({
                'mac': client['mac'],
                'description': client.get('description', 'Unknown'),
                'ipv6_global': client.get('ip6', 'N/A'),
                'ipv6_local': client.get('ip6Local', 'N/A'),
                'ssid': client.get('ssid', 'Wired'),
                'vlan': client.get('vlan', 'N/A')
            })

    return ipv6_clients

def check_ipv6_coverage(network_id):
    """Report IPv6 adoption rate."""
    url = f"{BASE_URL}/networks/{network_id}/clients"
    response = requests.get(url, headers=headers, params={"timespan": 3600})
    all_clients = response.json()

    total = len(all_clients)
    ipv6_count = sum(1 for c in all_clients if c.get('ip6'))

    print(f"Total clients: {total}")
    print(f"IPv6 clients: {ipv6_count} ({100*ipv6_count//total if total else 0}%)")

if __name__ == '__main__':
    clients = get_clients_with_ipv6(NETWORK_ID)
    for c in clients:
        print(f"{c['description']:30} | IPv6: {c['ipv6_global']:40} | SSID: {c['ssid']}")
    print(f"\nTotal IPv6 clients: {len(clients)}")
    check_ipv6_coverage(NETWORK_ID)
```

## Meraki MR Access Point IPv6 Features

```
MR Access Points (Cloud Managed):
- Transparent L2 bridge: Pass-through RA and DHCPv6 automatically
- No local IPv6 configuration needed on APs
- IPv6 management of the AP itself:
  Dashboard > Wireless > Access Points > [AP Name] > Status
  Shows AP management IP (IPv4 only in most firmware versions)

IPv6 SSID considerations:
- Isolated SSID: Block IPv6 RA spoofing between clients
- Guest SSID: Use separate IPv6 prefix for isolation
  Dashboard > Wireless > SSIDs > [SSID] > Firewall & Traffic Shaping
  Layer 3 Firewall: Block intra-client IPv6 traffic
```

## Firewall Policy for IPv6 on Meraki

```
Dashboard > Security & SD-WAN > Firewall

IPv6 Rules (under L3 Outbound rules):
Rule 1: Allow ICMPv6
  Protocol: ICMPv6
  Action: Allow

Rule 2: Allow DHCPv6
  Protocol: UDP
  Dst Port: 546-547
  Action: Allow

Rule 3: Allow DNS over IPv6
  Protocol: UDP
  Dst Port: 53
  Action: Allow

Rule 4: Allow established outbound
  Action: Allow (stateful - handled automatically)
```

## Troubleshoot Meraki IPv6

```bash
# SSH into MX (if enabled) or use Dashboard Tools > Ping

# From Dashboard > Tools > Ping
# Target: 2606:4700:4700::1111 (Cloudflare DNS IPv6)
# Interface: WAN1

# Check IPv6 routes on MX
# Dashboard > Security & SD-WAN > Route Table

# Packet capture for IPv6 debugging
# Dashboard > Network-wide > Packet Capture
# Filter: ip6
# Interface: WAN or LAN

# Check event log for IPv6 issues
# Dashboard > Network-wide > Event Log
# Event type: IPv6 DHCP
```

Meraki IPv6 deployment is largely automated through the Dashboard—enabling IPv6 on VLAN interfaces triggers automatic RA generation and DHCPv6 configuration. MR access points transparently bridge all ICMPv6 and DHCPv6 traffic, requiring no AP-specific IPv6 configuration for wireless clients to receive global IPv6 addresses.
