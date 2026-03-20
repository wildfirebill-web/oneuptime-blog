# How to Configure Infoblox for IPv6 IPAM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Infoblox, IPAM, DDI, Enterprise Networking

Description: Configure Infoblox DDI for IPv6 address management including network views, IPv6 networks, DHCPv6 ranges, DNS AAAA records, and automation via the Infoblox REST API (WAPI).

## Introduction

Infoblox provides integrated DDI (DNS, DHCP, IPAM) with comprehensive IPv6 support including DHCPv6 server, DNS64, AAAA record management, and automated address discovery. This guide covers Infoblox IPv6 configuration via the WAPI REST API.

## Step 1: Configure IPv6 Network View

```bash
# Create a network view for IPv6 (via WAPI)
curl -u admin:password \
    -H "Content-Type: application/json" \
    -X POST \
    "https://infoblox.example.com/wapi/v2.12/networkview" \
    -d '{
        "name": "ipv6-production",
        "comment": "Production IPv6 address space"
    }'
```

## Step 2: Create IPv6 Networks

```python
#!/usr/bin/env python3
# infoblox_ipv6.py

import requests
import json

WAPI = "https://infoblox.example.com/wapi/v2.12"
AUTH = ("admin", "password")
VERIFY_SSL = False

def wapi_post(endpoint, data):
    resp = requests.post(
        f"{WAPI}/{endpoint}",
        json=data, auth=AUTH, verify=VERIFY_SSL
    )
    resp.raise_for_status()
    return resp.json()

def wapi_get(endpoint, params=None):
    resp = requests.get(
        f"{WAPI}/{endpoint}",
        params=params, auth=AUTH, verify=VERIFY_SSL
    )
    resp.raise_for_status()
    return resp.json()

# Create IPv6 network (parent /48)
wapi_post("ipv6network", {
    "network": "2001:db8:0001::/48",
    "comment": "HQ Site IPv6",
    "extattrs": {
        "Location": {"value": "Headquarters"},
        "Environment": {"value": "Production"}
    }
})

# Create /64 subnet within the /48
wapi_post("ipv6network", {
    "network": "2001:db8:0001:0001::/64",
    "comment": "HQ Servers VLAN",
    "extattrs": {
        "VLAN": {"value": "10"},
        "Purpose": {"value": "Servers"}
    }
})
```

## Step 3: Configure DHCPv6 Range

```python
# Create a DHCPv6 range within the /64
wapi_post("ipv6range", {
    "network": "2001:db8:0001:0001::/64",
    "start_addr": "2001:db8:0001:0001::1000",
    "end_addr": "2001:db8:0001:0001::ffff",
    "comment": "HQ Servers DHCPv6 pool"
})
```

## Step 4: Create AAAA Records via WAPI

```python
# Create AAAA record in DNS
wapi_post("record:aaaa", {
    "name": "server-01.example.com",
    "ipv6addr": "2001:db8:0001:0001::10",
    "view": "external",
    "comment": "Web server IPv6"
})

# Create AAAA with PTR record
wapi_post("record:aaaa", {
    "name": "api.example.com",
    "ipv6addr": "2001:db8:0001:0001::20",
    "create_ptr": True,   # Automatically create PTR record
    "view": "external"
})

# Search for AAAA records
aaaa_records = wapi_get("record:aaaa", {
    "name~": "*.example.com",
    "_return_fields": "name,ipv6addr"
})
for record in aaaa_records:
    print(f"  {record['name']}: {record['ipv6addr']}")
```

## Step 5: IPv6 Address Allocation via WAPI

```python
# Allocate next available IPv6 address in a network
def allocate_next_ipv6(network: str, hostname: str) -> str:
    result = wapi_post(f"ipv6network?network={network}/_nextavailableip", {
        "comment": f"Auto-allocated for {hostname}",
        "num": 1
    })
    return result["ips"][0]["ip_address"]

# Allocate address and create AAAA record together
new_ip = allocate_next_ipv6("2001:db8:0001:0001::/64", "db-server-05")
wapi_post("record:aaaa", {
    "name": "db-server-05.example.com",
    "ipv6addr": new_ip,
    "create_ptr": True
})
print(f"Allocated {new_ip} for db-server-05")
```

## Step 6: IPv6 Network Discovery

```python
# Configure discovery for an IPv6 network
wapi_post("discovery:task", {
    "network": "2001:db8:0001::/48",
    "discovery_method": "ICMP6",
    "scan_interfaces": True,
    "tcp_scan_technique": "SYN"
})
```

## Step 7: DNS64 Configuration

```python
# Configure DNS64 for IPv4-only clients
wapi_post("dns:dns64synthesisgroup", {
    "name": "dns64-main",
    "prefix": "64:ff9b::/96",
    "comment": "DNS64 for IPv6-only clients reaching IPv4 services",
    "mapped": [{"address": "0.0.0.0", "prefix": "0"}]
})
```

## Conclusion

Infoblox provides enterprise-grade IPv6 IPAM through its WAPI REST API with atomic DDI operations — a single API call can allocate an address, create an AAAA record, and create a PTR record. The `_nextavailableip` endpoint for IPv6 networks automates address allocation without conflict checking. Use Infoblox's DNS64 feature when deploying IPv6-only environments that still need to reach IPv4-only external services. The extattrs (extensible attributes) system enables custom metadata like VLAN ID, environment, and owner tracking on IPv6 prefixes.
